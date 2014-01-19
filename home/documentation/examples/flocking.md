---
layout: default
title: Parallel flocking algorithm
permalink: /flocking.html

num: 40
outof: 50
partof: examples
description: This example shows how to render a highly parameterizable flocking algorithm set in parallel
image: flocking-icon.png
---
#A flocking algorithm
A flocking algorithm represents the interaction between n individual agents that obeys simple rules and observe a general behaviour emerging from it. This is particulary accurate in modelling a group of birds.

Each agent obeys 3 rules :
	
* Short range repulsion : The individual agent avoids the agents that are too close to him.
* Long range attraction : The individual agent is attracted by the agents that are far from him.
* Same heading : The individual agent tries to stay toward average heading of his neighbours.

If we configure a big number of agents obeying to these three rules, we will see a collective behaviour that can be different depending of the coefficients of the three rules, i.e. what is “close”, “far”, or by how fast the agent is attracted, repulsed or respects the average heading, etc.. -

The aim of this algorithm is to have a visual representation of this behaviour.

<div class="imageframe-deep">
  <img src="{{ homedir }}/resources/images/flocking_example.png"/>
</div>

#Introduction

The initial and naïve algorithm is quite simple : we just need to compare each of the agents to all the others, check if it is in the short or the long range and apply all our rules accordingly. To do this efficiently, we can simply apply the short and long range repulsion rule to the average coordinates of the birds  in the short and long range.
This is a `O(n^2)` time (by round) algorithm. In our implementation, we improve it with the use of a quadtree, which has a `O(n * log(n) * k)` execution time (with `k` the average number of birds that are un the long range of a bird) for each step. This is fine to test our parallel tools.
Since each bird has to be compared with all birds near it, we can execute each computation of new coordinates in parallel by giving to each instance the current set of birds and the bird to update.

For this, we need a Scheduler to execute the operations in parallel. We initialise a new one each time we enter in the parallelization mode.

    schedulerForToPar = 
    new Scheduler.ForkJoin(new Scheduler.Config.Default(numberOfProcessorsUsed))

Then we can parallelise our operation in an easily way by adding a .toPar to the data structure : 

    def launch()(implicit sch: Scheduler){
      if (useToPar){
        arrayBirds = 
	((arrayBirds.toArray.toPar).map(cycleForOneBird(arrayBirds, quadtree, _))).seq
      }
      else{
        arrayBirds = arrayBirds.map(cycleForOneBird(vectorBirds, quadtree, _))
        if (usePar && !parChanged)
          arrayBirds.asInstanceOf[ParSeq[Bird]].tasksupport = tasksupportForPar
      }
    }


#Graphic interface
When the program is launched, the user is invited to tick the box "play". 

It is possible, at any moment, to choose the parallelization mode, and the number of processors. 

There are several modes to be activated or deactivated, and some of the parameters values can be changed. All the available parameters are modifiable from the sliders or by typing the desired value on the corresponding text box and pressing enter.

If the number of birds involved change, all the current birds will be erased to be replaced by a new set of birds. This value correspond to the value `Flocking.numberOfBirds`.

The values on the column "range" correspond to the values of `Flocking.shortRange` and `Flocking.longRange`, and the values on the column coefficient correspond to the values `Flocking.coeffShort` multiplied by `-10` and `Flocking.coeffLong` multiplied by `100`. These numbers are multiplied because it was much more practical for the user to have int values to change instead of doubles.

The coefficient of moving forward correspond to the value `Flocking.coeffMovingForward` multiplied by `10`.
The coefficient of turning correspond to `Flocking.coeffturn` multiplied by `100`.
The maximum speed correspond to `200` minus the value `Flocking.minSpeed`.

The minSize of the quadtree correspond to the value `Flocking.minSizeQuadTree`.

#Anaysis of the implementation

##The class Bird

First of all we have to define the class representing the individual agent. 
This class is called `Bird` and have just for parameters his coordinates and those of his oriented vector. It has a supplementary internal value that represent his `Polygon` that will be used to draw it on the screen. This value is computed and updated in the `Graphic` object that we will see bellow.

##Object Graphics

The object `Graphics` represents the complete graphic interface. He has 4 parameters that the user can change, but carefully : the height and the width of the birds ground, the size allocated for the buttons interface (but note that the buttons in Java have a fixed size, if this value is changed, the sizes of the buttons have also to be changed accordingly), and the background color of the ground.
To update the visual representation, we just need to change it's value `Graphics.arrayBirds` and call the `repaint()` method. 


###init()
In the method `init()`, we declare all the components of the java swing interface. We initialise the windows with a `JFrame`,  we link them to the corresponding environment variables located in the `Core` object with listeners, we unify them in a `JPanel`, `jpanelButtons`, and add it to the `JFrame`. The ground is represented by the graphics of the object `Graphics` itself, that we also add in the `JFrame`. 

###drawPolygons(g: Graphics, bird: Bird): Bird
The method `drawPolygon` draws a polygon in the graphics object of Graphics. This method is always called sequentialy to avoid concurrent drawings.

###updatePolygons(g: Graphics, bird: Bird): Bird 
This method updates the component `Polygon` of a bird with a triangle that fits his current values. Indeed, the orientation and position values of a bird can change over time, so he need to have his representative polygon updated.

###drawQuadtree(g: Graphics, quadtree: Quadtree)
This method draws the current quadtree recursively.

###paintComponent(g: Graphics) 
This method is called with the method `repaint()`. We first erase all the old birds  drawing a new background rectangle, then we recompute the birds's polygons with the new values with `updatePolygons`. Depending of the parallelization mode, the operation will be done sequentially or in parallel. This is possible because each bird's polygon can be computed independently. Finally, we draw them with `drawPolygons`. 


## The class Quadtree
This object represents the quadtree interface. You can learn about the quadtree data structures [here](http://en.wikipedia.org/wiki/Quadtree).

On initialisation the quadtree is constructed recursively by putting each of the bird in the south west, south east, north east or north west square according to his coordinates. We have a different `List` of birds for each square, and we add each bird to the correct list. Then, we construct a quadtree for each of these subsquares.

###getBirdsIn(x1In: Int, y1In: Int, x2In: Int, y2In: Int): List\[Bird\]
Gets us the birds that are in the rectangle defined by the coordinates in argument. For each of the four subquadtrees of the squares, we analyse the part of the rectangle that is in this subquadtree, and  call recursively this function. We return the fusion of all the results for the four subquadtrees.



##The Flocking object :
This object is the place where the computations are done. 

###Parameters
The object begins with a set of constants that can be updated manually (not recommended) or in the graphic interface. If you update it manually, you have to respect  the maximum and minimum values of the graphic interface, otherwise you will cause a `IllegalArgumentException`.
These values are : 

<b>shortRange</b> : Defines what it is considered, in pixels, to be the radius of the circle in which all other objects will be considered to be close. The values for this parameter are from `1` to `20`.

<b>longRange</b> : Defines the same that shortRange but for the objects that will be considered as far. The objects that are not in the LongRange will not be considered, because they are too far to enter in interaction with our current object. The values for this parameter are from `1` to the `width` of the panel.

<b>coeffShort</b> : Defines the coefficient of attraction that will affect the birds that are in the short range. Since we want it to be repulsive (to respect the flocking algorithm), we define it as a negative value.
The values for this parameter  are from `-8.0` to `-0.1`.

<b>coeffLong</b> : Defines the same that `coeffShort`, but for the long range. This value is positive, because we want it to be attractive, and usually the absolute value is smaller that the `coeffShort` one, because the attractive forces would outdo the repulsive ones and all the objects would be attracted in one point. The values for this parameter are from `0.01` to `1.00`.

<b>coeffMovingForward</b> : The birds are always moving forward with respect to their orientation. Depending to this coefficient, they will move with more or less strength. It is better to have it superior to the `coeffShort` and `coeffLong`, because we would have birds that would go backwards. The values for this parameter are from `0.1` to `5.0`.

<b>groupMode</b> : If this boolean is set to true, we add a simple 4th rule, that specifies that the bird tend to turn to the average position of the birds that are in his short range. We call it the group mode because this creates groups of birds. Indeed, if a bird goes too far from the group, he tends to turn back to it.

<b>walls</b> : This boolean stipulates if there is walls on the borders of the window. If it is set to true the birds bounce on the borders, and if not the birds re-appear on the other side.

<b>coeffTurn</b> : When a bird has to turn, either to be in the average heading of his neighbours or to join the group in the group mode, it wouldn't be a good simulation if it did go directly to the right direction, because it would take the same time to turn short angles and big angles. That is why we define a coefficient of turning, between `0.01` and `1.00`, that will indicate the ratio of turning. Each time a bird wants to turn some angle, it will only turn this angle multiplied by this coefficient (at each turn).

<b>useQuadtree</b> : Specifies if we want to use the quadtree optimisation or not. This normally improves the algorithm when birds are far apart, but tend to slow when they are close to each other, because the birds the time used to compute the quadtree isn't in that case compensated by the time gained.

<b>drawQuadtree</b> : Specifies if we want to draw in real time the quadtree in the graphic interface.

<b>minSizeQuadTree</b> : Specifies the size in pixels of the tiniest possible larger part of a rectangle in the quadtree. This can go from `1` to the `width` of the window, but efficient values are usually between `1` and `10`.

<b>numberOfBirds</b> : Specifies the number of birds that are on the area.

<b>usePar</b> : Boolean that specifies if we have to use the classic parallel collections.

<b>useToPar</b> : Boolean that specifies if we have to use the workstealing tree parallel collections. It is impossible to have `usePar` and `useToPar` both set to true.

<b>minSpeed</b> : The minimum time, in milliseconds, that have to take each round at least. This value is necessary because is some cases with a small number of birds the iterations can go really too fast to be seen to the naked eye. The recommended values are from `0` to `200`, superior values would cause the program to behave too slowly.

<b>numberOfBirdsChanged</b> : Only necessary for the graphic interface and the initialisation, it is automatically set to true when the user modifies the number of birds present that he wants on the field. This causes the program to recompute a new set of birds

<b>parChanged</b> : Also only necessary for the user interface and the initialisation, specifies if the user changed the parallel mode, meaning the usage of classic, parallel, or workstealing tree collections or the number of processors to be involved in the computation.

<b>numberOfProcessorsUsed</b> : Number of processors to be used if we are using one of the two parallel modes.

<b>sizeBase and sizeLong</b> : The size in pixels of the base and the altitude of the triangles representing the birds in the graphic interface.

###isInthatDistance(flock1: Bird, flock2: Bird, distance: Int): Boolean : 
Tells if two birds are separated by no more that distance pixels computing their distance with the Pythagorean formula.

###moveThatBirdTo(positionX: Int, positionY: Int, coefficient: Double, flock: Bird)

This method  is used to update the position of a bird to a certain position. It will update the position of the bird to this direction according to a certain coefficient. This will be used to do the short range repulsion, the long range attraction and the moving forward step.
To compute it, we simply compute the direction where the bird need to go by computing the difference of their coordinates, and multiply it by their ratio to make sure that the bird will move less to this direction if it is farther :

    newCoordinates = coefficient * difference of coordinates * ratio of coordinates


###updateBorders(flock: Bird): Bird
This will take care of the borders of the screen. 
Depending if there are walls or not, it will update the bird differently. 
If there is, the orientation is updated by multiplying by `-1` one of the bird's direction oriented vector, to make him turn by an angle of `90°` in the correct direction (the only possible direction). We also make sure that if the bird already advanced a certain distance, this distance is reported on the other direction.

    newOrientation = current orientation +/- 90°
    newCoordinates = actual border +/- distance to be reported

In the other case, we just make one of his coordinates changes from one side of the screen to the other. We also make sure that the distance advanced is reported on the other side.

    distance to be reported = (old coordinate - current border of the screen)
    newCoordinates = other side of the screen +/- distance to be reported


We also always make sure that the bird is always at `sizeLong` pixels  distance of the screen, so that we never have the front side going out of the screen, which wouldn't be realistic.

###updateOrientation(flock: Bird, averageAlignmentX: Double,  averageAlignmentY: Double): Bird 
This method will turn our bird to the direction given by the two other arguments. We don't want to turn it directly, as seen in the definition of the `coeffturn` parameter . So to do this, we substract to the birds direction oriented vector the difference of this vector and the new coordinates multiplied by the turning coefficient : 

    newCoordinates = oldCoordinates - (newCoordinates – oldCoordinates) * turning coefficient

But this will give us a non normalized vector. Since we need a directional vector, we normalize it by dividing it's two coordinates by his norm, computed with the Pythagoras theorem :

    norm = sqrt(y^2 + x^2)
    newCoordinates = newCoordinates / norm


###cycleForOneBird(flocks: GenSeq\[Bird\], quadtree: Quadtree, flock:Bird): Bird
This is the main class of the Core object. It will compute, for a given bird his new position using the list of the other birds. If we are in quadtree mode, we reduce the list of birds to the ones the quadtree gives us.
Since this method is meant to be used in parallel, we created a deep copy of the bird to do our computations. Then, we compute the average position of the birds that are in the short range, and for the ones that are in the long range. We find this with the function isInthatDistance. We also sum all the position vectors of the birds in the short range, which give us a bigger vector that represents the average orientation when we normalise it. 
We take advantage of this computation to shift birds  that are exactly in the same position, to avoid having multiple birds represented only by one triangle (their superposed triangles) in the graphic interface.
We use the function moveThatBirdTo to move, using the correct coefficients, the bird forward (to a point that we compute that is just before the bird), and to the averages positions of the short and long range.
We use the function updateOrientation to turn the bird to the average orientation vector.
We use the function updateBorders to take care of the borders.

###launch()
This is the function where the method cycleForOneBird is called. We map our arrayBirds with this function to create a new arrayBirds with updated positions and orientations. Depending on the parallelization mode, this operation will be performed sequencially or in parallel with classic collections or workstealing tree. This is possible because each bird only needs the previous list of birds, and potentially the quadtree reprensenting this previous list, so each operation can be performed independently of the others.

#Main and parallelization

##Main function
The main function takes care of the calling of functions (sequentially or  in parallel), the generation of birds, the generation of a new quadtree for each iteration in quadtree mode, the updating of the graphic visualisation, and the measurement of execution time.

We are going to maintain a while loop that is going to indefinitelly execute what we are going to call steps : 

First we check if the variable `numberOfBirdsChanged` is set to true. If so, then the program just began or the number of birds has been changed by the user, so we create a new Array\[Birds\], that we store in `Flock.arrayBirds`, with positions generated pseudo-randomly : with the same number of birds will appear the same set of birds, and draw it on the screen with Graphics.
Then we check if the variable `parChanged` is set to true. If so, like for `numberOfBirdsChanged`, we have to take care of the new parallelization parameters.

If we are in quadtree mode, we create a new quadtree with the current position values that will be passed in argument for `cycleForOneBird`.
Finally, we call the function `launch()` to start modify our Array of birds.

##Testing the parallel collections
To test the performance of the workstealing tree parallel collections, try to choose your desired parameters, or just keep the default ones, and choose the classic parallel collections mode. Make the program run something like one minut or two by checking the play box, and stop it. Remember the average execution time, and now choose the workstealing parallel collections mode, and press the restart button. 
Wait the same amount of time, stop the program, and compare the execution times. Normally, the new parallel collection will perform better.
The best is obviously to run two instances of the program and make them run at the same time, to avoid low level bias.

You can find the complete code for this example [here](https://github.com/scala-blitz/scala-blitz/blob/master/src/test/scala/org/scala/optimized/test/examples/Flocking.scala).

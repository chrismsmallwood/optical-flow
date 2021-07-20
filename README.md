

# Project 2 - Optical Flow

## Overview:

The goal of the project is to demonstrate optical flow using the Lucas-Kanade method and the Horn-Schunck method. Optical flow is a way to estimate 3D motion in a real world scene based on 2D pixel displacement in images captured of the scene, those images being separated by a time step.  This projection of motion from 3D to 2D means there is some loss of information so our optical flow estimates will not be perfect representations of the 3D motion.  I will explore these two methods for determining optical flow and compare their performance.

Both Lucas-Kanade and Horn-Schunck methods share several commonalities.  The input for both methods will be two images separated by a small time step and the output will be two matrices, u and v. These output matrices encode the vector values of optical flow for the corresponding pixel locations in the input. When we combine the values of u and v at every index into a single vector and then plot those vectors on top of the input, we can visualize the optical flow of the input. Both methods depend on several assumptions:

The camera is static throughout the time step between capturing the images.  
The time step is relatively small and therefore the pixel displacements are relatively small. Ideally sub-pixel level displacements show the best results. 
There is a brightness constancy of the pixels between the two images. This means that corresponding points in the two images will retain the same brightness value from the first image to the second image.  This a strong assumption for real world images because lighting conditions, surface materials, etc can greatly influence the brightness value of the same point on an object when that object moves. 
Pixels nearby to one another will move in similar directions and thus have similar optical flow.

The first two assumptions apply equally to both methods, but the last two assumptions are where we some some variation between Lucas-Kanade and Horn-Schunck.  Next I will briefly explain the two methods and how I implemented them.

## Lucas-Kanade: 

This method takes both brightness constancy and nearby pixel displacement as totally constant. Using the concept of brightness constancy, we can derive the following equation:  .
The equation simply means that changing the locations of pixels by u and v will cause zero change in the intensity value of the pixel.  The method leverages this assumption along with the assumption that neighborhood pixels will have the same displacement (u, v) in order to solve for u and v at the target pixel location in an overdetermined system. 

The steps in my implementation:
Input X-derivative (), Y-derivative () and time derivate () images along with a neighborhood (patch) size parameter.
For every pixel in the image, setup a matrix A and a vector b.
For every pixel in the current patch, fill matrix A with  values and vector b with  values from the corresponding locations in the derivative images.
Set up a linear system of Ax = -b where x = [u, v].
Solve this overdetermined system using linear least squares to find [u, v] for the target pixel.
Record values u and v in their output matrices at the location corresponding to the target pixel.

Controlling the results of Lucas-Kanade comes from our choice of patch size.

## Horn-Schunck:

This method relaxes both brightness constancy and the nearby pixel displacement constraint.  Instead of assuming neighboring pixels will have the same displacement (constant flow), it assumes they have similar displacements (smooth flow).  It does so via an energy function with the goal of minimizing the energy function to find the best result.  The energy function is:   .
Where  is a brightness constancy equation and  is a smoothness equation.  The smoothness term is weighted by a parameter lambda which is used to control the effect of one term versus the other in the result. We minimize this energy function via gradient-descent which involves finding partial derivatives of the function with respect to u and with respect to v then deriving update equations from the partial derivatives.  For each iteration of gradient descent, we take a step towards the optimal solution to the energy function and hence the most accurate values of u and v. 

The steps in my implementation:
Input X-derivative (), Y-derivative () and time derivate () images along with a lambda parameter and a max number of iterations.
For every pixel in the image, compute estimated values of u and v at that pixel via the pixels directly above, below and on both sides.
Compute the gradient descent step via the update equation.  This equation utilizes , , ,  and the estimates for u and v computed in the previous step.
Subtract this gradient descent step for both u and v from the estimated values of u and v in order to determine our current value for u and v.
Repeat this process until u and v converge to their optimal values.

Controlling the results of Horn-Schunck comes from our choice of lambda.


## Experiments:

1.Lucas-Kanade on Synthetic Spheres

The images on the right show the outputs of Lucas-Kanade on the sphere images when parameterized with different patch sizes.  I found that increasing the patch size improved the accuracy of the vectors within the sphere with the side effect of decreasing accuracy of some points in the background surrounding the sphere. In the left column, you can see that as we use larger patches, the vector field becomes more uniform in direction, thus better informing the sphere’s motion. This is the benefit of a larger patch.

The downside of background inaccuracy is easily explained by patch size. Once we have a sufficiently large patch, target pixels in the static background near the sphere’s edge will have a number of sphere pixels that fall within the patch and thus non-zero values of (u,v) will be calculated for that background pixel. 

The images showing magnitude of optical flow in the right column demonstrate that increasing patch size also increases the accuracy of the size of the optical flow vectors to an extent. In these images lighter color means greater vector magnitude. You can observe that larger patches show greater disparity in vector magnitude between parts of the sphere that have little motion, like the top, and parts of the sphere which have greater motion, like the middle and bottom.  Also, we can even better visualize the side effect of background pixels increasingly straying from the desired zero magnitude as patch size increases. With larger patches the area surrounding the sphere has larger vectors and the edge of the sphere becomes less discernible. 

My takeaway is that the choice of patch size is an important factor in determining optical flow.  The choice will ultimately be determined by the larger goals of the application using optical flow. There is a tradeoff between the accuracy of the vector directions of the moving objects and the blurring of the edge of the moving objects with the background.  Increasing the patch size will increase the direction accuracy within the object, but decrease accuracy near the edges.  The user will need to decide what is most important for their application when choosing this parameter for Lucas-Kanade.


2. Horn-Schunck on Synthetic Spheres

The images on the right show the outputs of Horn-Schunck on the sphere images when parameterized with different lambda values. As mentioned previously, lambda controls how much weight we give to brightness constancy versus smoothness in the overall energy function.  High values of lambda give priority to smoothness and low values of lambda (below zero) give priority to brightness constancy.  With the synthetic sphere data, the brightness constancy can be heavily leveraged because the corresponding pixels between the two time stepped images will retain exact brightness constancy.  This is why we see a nice uniform vector field at lambda of 0.1.  However, in real images, brightness constancy cannot be guaranteed in the same way, even when the time step is small.  Therefore, weighting the smoothness component over the brightness constancy will often yield better results. 

An interesting effect can be seen in the magnitude images where larger values of lambda result in larger vector magnitudes near the top of the sphere, where motion is presumed to be relatively small.  This means that giving a higher weight to smoothness can skew the magnitude of optical flow. 

Overall, I would say that a lambda of one, meaning the brightness and smoothness terms are weighted equally, produces the best results for these images.  We can observe uniform vector direction, relative magnitudes where we would expect them and a minimum of inaccuracy near the edges. My takeaway is that the choice of lambda does not have a huge effect on the results in the case of these synthetic sphere images. However, I think the choice of lambda is highly consequential in real world images that are much noisier and less precise.  I will explore this prediction further with the real traffic image experiments shown below.


3. Lucas-Kanade on Real Traffic

The image below is the best result I obtained using Lucas-Kanade on the real traffic images.  This result comes from a patch size of 15 by 15 without blurring or downsampling the input images. 


Obtaining good results using Lucas-Kanade and Horn-Schunck was most difficult because of factors involving computing the derivative images.  The derivative images used are shared between the methods, but I will discuss this factor in more detail in the last section of the project.  Given derivative images that are well computed, obtaining good results boils down to the choice of patch size along with decisions about blurring and downsampling.  This process is simplified with good decisions about what values to begin experimenting with first.  Fortunately the synthetic sphere experiments informed these decisions. 

My process for obtaining these results was to compute vector field and magnitude images for the same range of patch sizes as the synthetic sphere experiments (n = 3, 5, 11, 21) and use these as a basis for fine tuning the patch size.  I already suspected from the synthetic sphere experiments that larger patch sizes would produce better results and the findings supported this hypothesis.  The best results came from patch sizes of 11 and 21, but I noticed that at n=21 not only were the edges of the vehicles heavily blurred but there was also a decent amount of background showing noticeable magnitude.  From there I simply tuned to find this result which balances capturing vehicle motion with minimizing spurious vectors near the vehicle edges and in the background. I also played around with different combinations of blurring and downsampling the image, but found that they gave little improvement to the output images at this patch size.


4. Horn-Schunck on Real Traffic

The image below is the best result I obtained using Horn-Schunck on the real traffic images.  This result comes from a lambda value of 500 without blurring or downsampling the input images.



Obtaining good results with Horn-Schunck was comparable to with Lucas-Kanade.  I began by computing the vector images using the same lambda values tested on the synthetic spheres ( = 100, 10, 1, 0.1).  However, my hypothesis was that choices of lambda in Horn-Schunck would translate less cleanly between synthetic and real world data than what I experienced with choosing patch size for Lucas-Kanade .  I was predicting that giving a heavier weight to the smoothness term would produce better results on the traffic images than it did on the spheres and this was confirmed by the experiments.

What I found was that a lambda value of 100 produced the best results from the beginning. From there I scaled lambda even higher until I found satisfactory results.  I found that pushing lambda above a value of 500 caused little change in the output images so this was the value I decided to settle on.  I believe this difference between choices of lambda on synthetic data versus real world data comes from the unreliability of brightness constancy in real world data.  We can ensure brightness constancy in synthetic data, but this must be relaxed when dealing with real world data, even small changes in position can result in different intensity values of corresponding points.  On the other hand, both synthetic and real images share the feature that nearby pixels will often move smoothly so giving the smoothness term priority in the algorithm will highlight objects in the image that are moving cohesively.  


## Summary:

5.  Overall Thoughts on Both Methods

Comparing the methods in terms of efficacy, I would say that the Horn-Schunck method showed better results overall.  Both methods can be used to effectively demonstrate optical flow especially when the data is clean.  We saw this with the synthetic sphere experiments, both methods were able to capture a great representation of both the direction and magnitude of the spheres movement once parameters were tuned. However, I was happier with the results of the Horn-Schunck method on the real traffic images.  I found that with Lucas-Kanade there was always a trade off between capturing the movement of the vehicles and showing significant background or edge case movement.  Using Horn-Schunck, I was able to better tune the parameters to pick up on only the movement of the vehicles and little else.  I suspect this advantage results in Lucas-Kanade having stricter restraints on brightness constancy and constant flow versus smooth flow.

Similarly, I found that choosing parameters for Lucas-Kanade was more difficult on the real traffic images than with Horn-Schunck.  With Horn-Schunck, I already had an idea of whether weighting smoothness or brightness more heavily would produce better results simply from examining the input images.  In contrast, I had little idea from the beginning as to what size patch would produce good results with Lucas-Kanade.  I guessed that it would not be a very small patch, but besides that I simply had to experiment with many sizes.

A downside of Horn-Schunck is that is took longer to run than Lucas-Kanade.  Lucas-Kanade only iterates through every pixel in the image once, and does some calculations using every neighbor pixel of the target pixel.  Runtime for Lucas-Kanade does slow down as patch size increases, but regardless, you are never iterating through the whole image more than once.  Additionally, there are methods for speeding up the process of computing neighborhood pixels.  In contrast, Horn-Schunck must iterate through every pixel in the image many times before it converges to an optimal result.  This is due to the nature of using gradient descent to solve for the output vectors, each iteration through the image gets us one step closer to a satisfactory result, but we must take many steps to reach that result.

In terms of implementation difficulty, I did not find one method significantly harder than the other.  As long as you have a grasp on the theory behind each step in the method, the implementation only requires a small number of commands.  What I struggled with most on this assignment was computing derivative images which would allow the methods to return decent results.  I somehow started out computing partial spatial derivatives using a Sobel filter of size 11, much too large to be a good input for the methods.  I also completely forgot to normalize the traffic image intensity values between -1 and 1 before computing the temporal derivative image.  This resulted in a temporal derivative that was extremely noisy in the background and barely captured the motion of the vehicles.  I spent much too long tuning and tweaking my already accurate algorithms and parameters to no avail before the Professor alerted me to the fact that the derivative images where the culprit.

In general I liked how the project built upon our theoretical knowledge of the methods from lecture.  We already had plenty of insight into how and why the methods worked and applying that knowledge to synthetic data first before real world data provided a nice transition.  

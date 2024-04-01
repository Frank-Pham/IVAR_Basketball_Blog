+++
title = 'Development Phase'
draft = false
+++

## Implementation: Locomotion

### Initial Prototype

One of the main restrictions during the development of the initial prototype for the locomotion technique was how the players movement is restricted due to the confined space, in which the player is situated in contrarily to the open court in basketball. Thus running up and down your room wasn’t an feasible option, so i kept the locomotion mechanics simple & straightforward: the way and the direction you dribble the ball dictates the players movement within the virtual space without the need to use your legs. Hence a forward dribble propels the player forward. In the initial prototype a downward motion (y-Axis) translates to a instant movement/teleportation: 

```csharp
  private void MovePlayer()
    {
        Vector3 rightControllerPos = OVRInput.GetLocalControllerPosition(rightController);
        float dribbleSpeed = (rightControllerPos - lastPosition).magnitude / Time.deltaTime;

        if (dribbleSpeed >= 1.0)
        {
            if (OVRInput.GetLocalControllerAcceleration(rightController).y >= 2.0)
            {
                isDribbling = true;
                Debug.Log("Dribbling:" + isDribbling);

                this.transform.position += new Vector3(dribbleSpeed, 0.0f, 0.0f);
                Vector3 forwardDirection = Camera.main.transform.forward *2;
                forwardDirection.y = 0.0f;
                forwardDirection.Normalize();

                this.transform.Translate(forwardDirection * speed * Time.deltaTime);
            }
        }

        lastPosition = rightControllerPos;
    }
```

The `MovePlayer` function basically checks if the velocity of the `OVRController` exceeds a certain threshold so it can move the player forward. My intention behind this code was to mimick a dribble motion by checking only the acceleration in the y-Axis. While it was effective to move around like this at first glance by spamming the dribble motion, it didn’t had the sense of intuitiveness and smoothness because the movement was decoupled from interacting with the basketball. Additionally there were major flaws like setting `forwardDirection.y = 0.0f` that came to haunt me later on at the part of the parkour where the slope begins (more on this later in Challenges). Another flaw with this Code is that you can’t control the dribble of the ball and you would lose control over the ball 9 out of 10 times. In addition to that, the instant translation of the forward movement made matters worse because you couldn’t orient yourself quick enough to locate and react to the movement of the basketball.

### Final implementation & adjustments

In the new `Dribble.cs`  Script, which is attached to Basketball `GameObject` , all the dribbling related logic is defined. There are few tweaks so the handling of the ball feels more intuitive instead of relying on the in-game physics and the reaction time of the player like beforehand:

![IVAR_dribbling](https://github.com/Frank-Pham/IVAR_Basketball_Blog/assets/58122562/6a55bfbc-13c4-4a97-a2b2-744da6fa4af3)

Now the player starts to gradually move towards the basketball whenever it hits the ground, giving the player more time to react to the ball.
```csharp
void Update()
    {
        if (ballInHand)
        {
            hitGround = false;
            CrossOver(parentHand.transform.position);

            Vector3 controllerVelocity = OVRInput.GetLocalControllerVelocity(currentController);
            Quaternion controllerRotation = OVRInput.GetLocalControllerRotation(currentController);

            float speed = controllerVelocity.magnitude;

            bool isDribbling = Vector3.Dot(controllerVelocity.normalized, controllerRotation * Vector3.down) > 0;
            Debug.Log("Speed" + speed);

            if (speed > dribbleThreshold && isDribbling)
            {
               ....
                Vector3 forwardDirection = controllerRotation * Vector3.forward;
                ballInHand = false;
                this.isDribbling = true;
                rb.AddForce(forwardDirection * 3 - Vector3.down , ForceMode.VelocityChange);
                rb.useGravity = true;
                rb.isKinematic = false;
	              ....
                magneticForce = 0f;
                lastControllerPos = currentControllerPos;

                StartCoroutine(TriggerHapticFeedback(currentController));
            }
        }

    }
```

Similar to the initial prototype, the script checks the velocity of the controller to initiate the dribble. Additionally to that there are two key additions, which are on the one hand `OnTriggerEvent` to keep track in which hand the ball is located and on the other the magnetic property of the ball.

```csharp
if (other.gameObject.tag == "hand")
        {
            magneticForce = 10f;
            Debug.Log("In collider hand");
            this.isDribbling = false;
            parentHand = other.gameObject;
            currentController = rightController;
            Vector3 rightControllerPos = OVRInput.GetLocalControllerPosition(rightController);
            ballInHand = true;
        }
```

The magnetic property of the ball, which kind of behaves like a Jo-Jo, ensures that the ball always returns to the controller which initiated the dribble motion (aka. the `parentHand`). Simply by subtracting the position of the controller and the position where the basketball hit the ground gets you the direction where the ball should return to. The last step is to add a force to the rigidbody of the ball and a `magneticForce` which defines how strong the magnetic force should be. 

So what is the thought process to have a magnetic force? It is highly unrealistic to have something like this in basketball one might think right? The reasoning behind this is two-fold :

- Player can control the basketball better
- Makes dynamically switch hands easier (f.e. to mimick a crossover move)
    
    ```csharp
     // Dribble.cs: CrossOver()
     if (hitGround && isDribbling) {
            Vector3 directionToHand = parentHand.transform.position - transform.position;
            rb.AddForce(directionToHand.normalized * magneticForce);
            rb.velocity *= dampingFactor;
        }
    ```
    

This eradicates one of the annoying flaws that was described beforehand and offers the player a new layer of flexibility when handling the ball. I added a damping factor to the velocity of the ball because the magnetic force on the basketball was too powerful sometimes.

Also the interaction between the `LocomotionTechnique.cs` and `Dribble.cs`  Scripts was the key to make the locomotion technique more intuitive and smooth. One of the solutions, that ChatGPT gave me, is to use linear interpolation `Lerp()`, which is already provided by Unity’s vast variety of built-in functions. I ended up moving over & adjusting the locomotion mechanics in the `LocomotionTechnique.cs` in `LateUpdate()` . The method initially checks if a target position exists for the player to move towards. This desired target is dynamically set based on game events, such as the location where a basketball has landed/collided like the floor and is getting tracked by the `dribbleManager` aka.`Dribble.cs`.

```csharp
// LocomotionTechnique.cs: LateUpdate()
if(dribbleManager.hitGround && !throwableManager.isPassing) {
  Vector3 desiredPosition = target.position;
  desiredPosition.y = dribbleManager.groundY; 
  transform.position = Vector3.Lerp(transform.position, desiredPosition, smoothSpeed * Time.deltaTime);
}
```

### A high-level description of how the scripts interact with each other:

1. Player pickups the ball (`OnTriggerEnter`) → `ballInHand = true`
    - Magnetic force is turned on → ball moves to the parent hand
2. In `Dribble.cs` : Player initiates dribble → Set `ballInHand = false` & `isDribbling = true`
    - Magnetic force is turned off → ball moves towards the ground
3. In `LocomotionTechnique.cs` : Check constantly `if(hitGround == true)` 
4. If `true` → Player moves to the position (linearly interpolated) where the ball hit the ground
    - Magnetic force is turned on → ball moves to the parent hand
5. Repeat 

Gameplay of the dribbling implementation: [here](https://drive.google.com/file/d/13oyQo1um4KoxBUTqsR70QFchnmZqEq4L/view?usp=sharing)

## Second Locomotion Technique: Dunking

I thought it would be pretty sweet to be able to dunk the basketball because IRL one might be limited through their physical capabilities. So how does this work? It is basically like an alley-oop that you throw to yourself and the player is able to initiate a short dunking teleportation to the throwing direction of the ball. Here is a quick instruction on how the dunking locomotion technique works:

1. Hold/Grab the ball with the right index trigger
2. Push forward like a pass while holding the right index trigger
3. Ball is in the air
4. While ball is in the air, quickly push controller upwards
5. Slam dunk

![IVAR_dunking](https://github.com/Frank-Pham/IVAR_Basketball_Blog/assets/58122562/d6979f02-d4e4-4c57-bae1-21520157b15a)

The dunking implementation consists of two phases and builds on top of the logic moving towards a desired position that we already seen in `LocomotionTechnique.cs` . 

### Phase 1: Throwing the Alley-Oop

For the throwing/passing mechanic I followed two tutorials from [Black Whale Studio](https://www.youtube.com/watch?v=yDplNn9HlS4) and [Lights&Clockwork](https://www.youtube.com/watch?v=jVmqMy5vusU). I ended up using mainly the throwing logic from the first tutorial because the latter one didn’t took into account the velocity of the controller, which resulted in a  powerful throw even though the throwing motion was slight.

```csharp
// Throwable.cs: Update()
if (speed > passThreshold && isPassingForward) {
    isPassing = true;
    Vector3 controllerPosition = OVRInput.GetLocalControllerPosition(OVRInput.Controller.RTouch);
    PassBasketball(controllerPosition, controllerRotation, controllerVelocity);
    lastPassTime = Time.time;
    isPicked = false;
}
```

Similar to the dribbling logic, where I checked if the player surpasses a certain threshhold in the downwards direction, the passing logic behaves the same but in the forward direction. If it is the case then `PassBasketball()` is called, which adds a force to the rigidbody of the basketball . Also set `isPassing = true` enables the second phase to dunk towards the ball.

### Phase 2: Dunk towards the ball

```csharp
if(isPassing == true) {
    Vector3 localVelocity = OVRInput.GetLocalControllerVelocity(OVRInput.Controller.RTouch);
    float upwardsVelocity = localVelocity.y;

    if(upwardsVelocity > dunkThreshold && Time.time > lastPassTime + cooldown)
        isDunking = true;
    }
}
```

After that I check if the ball is in the air (`isPassing == true`) and enable the dunk if the player moved the controller quick enough in the upwards direction. Additionally a cooldown is added to ensure that the player can’t spam the dunking motion. 

 In `LocomotionTechnique.cs`  it check constantly if a dunking motion is initiated (`isDunking = true`) similar to the dribbling implementation.

```csharp
// LocomotionTechnique.cs: LateUpdate()
else if(throwableManager.isDunking) {
  Vector3 desiredPosition = target.position;
  desiredPosition.y = target.position.y; 
  transform.position = Vector3.Lerp(transform.position, desiredPosition, smoothSpeed * Time.deltaTime);
}
```

Gameplay of the dunking implementation: [here](https://drive.google.com/file/d/1JB8GBmEefcU_0TlB7CNYqMMyxxcVA191/view?usp=sharing)

## Haptic & auditive feedback

To improve presence for the players, I added haptics and soundeffects. Every time the player initiates a dribble haptic feedback in form of controller vibration is activated.

```csharp
// Dribble.cs 
StartCoroutine(TriggerHapticFeedback(currentController));

private IEnumerator TriggerHapticFeedback(OVRInput.Controller controller)
    {
        OVRInput.SetControllerVibration(1, 1, controller);

        yield return new WaitForSeconds(hapticDuration);
        OVRInput.SetControllerVibration(0,0, controller);
    }
```

For this I followed [this](https://www.youtube.com/watch?v=qr-k3swrLQE) tutorial about Metas Haptics SDK and created a simple coroutine to enable & disable the haptic feedback. 

Additionally everytime the player dribbles the basketball and it hits the ground a “bouncing” sound effect is played.

### Visual feedback: Trail renderer

I also added a trail renderer to the basketball GameObject to give the player a visual feedback in which direction the ball is heading. The thought process behind this is to help visualize the trajectory of the ball when a dunking motion is initiated.

![trailRenderer](https://github.com/Frank-Pham/IVAR_Basketball_Blog/assets/58122562/f73c7ef1-6b17-44ad-b3a7-0d94c59b5dd1)

## Challenges

- Description: When the player reaches the slope part of the parkour. He/she is unable to move the slope upwards and moves into the slope instead which leads to seeing parts of the parkour that weren’t meant to be seen. The problem lies in `forwardDirection.y = 0.0f`
- Solution:

```csharp
private void OnCollisionEnter(Collision collision)
    {
        if (collision.gameObject.CompareTag("ground")) {
            hitGround = true;
            magneticForce = 10f;
            ContactPoint contact = collision.contacts[0];
            groundY = contact.point.y;
            this.GetComponent<AudioSource>().Play();

            Debug.Log("Ground Y: " + groundY);
        }
    }
```

After a lot of debugging and testing out new solutions, the solution to this problem was rather trivial. The `ContactPoint` class stores information about the contact point where a collision occurs. First I tagged all parts of the parkour with streets & the slope with a “ground” tag. Then when everytime the basketball hits the ground, I extract the y-position from the first contact point of the collision and set it to the new y-position of the player, which gets updated in `LocomotionTechnique.cs`. Now the y-position is calculated correctly when the player moves upward & downward the slope part of the parkour.

## Materials
To give the sphere a more realistic look, I added a basketball texture to a material and applied it to the basketball. The texture can be found [here](https://www.robinwood.com/Catalog/FreeStuff/Textures/TextureDownloads/Balls/BasketballColor.jpg).
![Untitled (4)](https://github.com/Frank-Pham/IVAR_Basketball_Blog/assets/58122562/298917fa-ec26-483b-a3fc-61b559d44328)

Furthermore I added a physics material to give the basketball a realistic bounce when it collides with other objects. Then i applied it to the sphere collider of the basketball GameObject. Here are the configurations of the physics material:
![Untitled (3)](https://github.com/Frank-Pham/IVAR_Basketball_Blog/assets/58122562/4bf7be17-9cf5-4919-bb71-b3d2692535d7)

## Interaction Technique: Spinning basketball
![IVAR_dribbling](https://github.com/Frank-Pham/IVAR_Basketball_Blog/assets/58122562/8f4e2d18-d55a-4138-9112-3b94d3466b48)

To stick with the basketball narrative, I used a spinning basketball as a metaphor to rotate the T-shaped object in the interaction task. My first attempt to approach this was to use Meta’s `ControllerHands` and Gesture detection to stick the ball into the T-shape and rotate it by doing a swipe motion. Sadly the use of `ControllersHands` in combination with Metas Interaction SDK is limited and more optimized for `Hands` tracking, so i ditched this idea. (Failed attempts can be seen in `**ControllerGestureDetector.cs`**  & `GestureDetector.cs`

So I came up with a simpler solution by using the hand trigger of the right controller which makes the right `ControllerHands` index finger point up (👆) so it mimicks the starting position if you try to spin a basketball IRL. Then if the player pokes the T-shaped object with the controller, a collision will be detected and the ball spawns into the T-shaped object and it sticks to the index finger of `ControllerHands` until the release of the hand trigger. The left controller then controls the speed of the rotation/spin of the basketball by moving it left or right on the x-Axis plane. If the movement is leveled, the spin slows down and the T-shape will be easier to place into position. The full implementation can be seen in `MyGrab.cs`. 

Gameplay of the spinning basketball implementation: [here](https://drive.google.com/file/d/1MQ35fk9pt6QXU9y1ok72Dc0TkjzG__SI/view?usp=sharing)

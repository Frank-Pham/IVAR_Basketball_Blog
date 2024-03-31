+++
title = 'Development Phase'
draft = false
+++

## Implementation: Locomotion

### Initial Prototype

One of the main restrictions during the development of the initial prototype for the locomotion technique was how the players movement is restricted due to the confined space, in which the player is situated in contrarily to the open court in basketball. Thus running up and down your room wasn’t an feasible option, so the first step is to keep the locomotion mechanics simple & straightforward: the way and the direction you dribble the ball dictates the players movement within the virtual space. Hence a forward dribble propels the player forward. In the initial prototype a downward motion (y-Axis) translates to a instant movement/teleportation: 

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

The `MovePlayer` function basically checks if the velocity of the `OVRController` exceeds a certain threshold so it can move the player forward. My intention behind this code was to mimick a dribble motion by checking only the acceleration in the y-Axis. While it was effective to move around like this at first glance by spamming the dribble motion, it didn’t had the sense of intuitiveness and smoothness I was looking for. Additionally there were major flaws like setting `forwardDirection.y = 0.0f` that came to haunt me later on at the part of the parkour where the slope begins (more on this later on in Bugs). Another flaw with this Code is that you can’t control the dribble of the ball and you would lose control over the ball 9 out of 10 times. In addition to that, the instant translation of the forward movement made matters worse because you couldn’t orient yourself quick enough to locate and react to the movement of the basketball.

So I had to came up with a better solution and refractor the code to fit 

### Final implementation & adjustments

In the new `Dribble.cs`  Script, which is attached to Basketball `GameObject` , all the dribbling related logic is defined. There are few tweaks so the handling of the ball feels more intuitive instead of relying on the in-game physics and the reaction time of the player like beforehand:

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
    - Magnetic force is turned off → ball moves toward the ground
3. In `LocomotionTechnique.cs` : Check constantly `if(hitGround == true)` 
4. If `true` → Player moves to the position where the ball hit the ground
    - Magnetic force is turned on → ball moves to the parent hand
5. Repeat 

## Second Locomotion Technique: Dunking

I thought it would be pretty sweet to be able to dunk the basketball because IRL one might be limited through their physical capabilities. So how does this work? It is basically like an alley-oop that you throw to yourself and the player is able to initiate a short dunking teleportation to the throwing direction of the ball. Here is a quick instruction on how the dunking locomotion technique works:

1. Hold/Grab the ball with the right index trigger
2. Push forward like a pass while holding the right index trigger
3. Ball is in the air
4. While ball is in the air, quickly push controller upwards
5. Slam dunk

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

 In `LocomotionTechnique.cs`  it check constantly if a dunking motion is initiated (`isDunking = true`) similar to dribbling.

```csharp
// LocomotionTechnique.cs: LateUpdate()
else if(throwableManager.isDunking) {
  Vector3 desiredPosition = target.position;
  desiredPosition.y = target.position.y; 
  transform.position = Vector3.Lerp(transform.position, desiredPosition, smoothSpeed * Time.deltaTime);
}
```

## Haptic & auditive feedback

To improve presence I added haptics and soundeffects. Every time the player initiates a dribble haptic feedback in form of controller vibration is activated.

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

### Trail renderer

I also added a trail renderer to the basketball GameObject to give the player a visual feedback in which direction the ball is heading. The thought process behind this is to help visualize the trajectory of the ball when a dunking motion is initiated.

![trailRenderer](https://github.com/Frank-Pham/IVAR_Basketball_Blog/assets/58122562/bfdb2f5c-ae6a-4f05-8514-dba6203a6683)

## Challenges

- Description: When the player reaches the slope part of the parkour. He/she is unable to move the slope upwards and moves into the slope instead. The problem lies in `forwardDirection.y = 0.0f`
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

After a lot of debugging and testing out new solutions, the solution to this problem was rather trivial. The `ContactPoint` class stores information about the contact point where a collision occurs. First I tagged all the streets on the parkour & the slope with a “ground” tag. Then when everytime the basketball hits the ground, I extract the y-position from the first contact point of the collision and set it to the new y-position of the player, which gets updated in `LocomotionTechnique.cs`.

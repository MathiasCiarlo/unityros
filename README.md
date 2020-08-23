# ROSBridgeLib
A Unity library for communication with ROS through [RosBridge](http://wiki.ros.org/rosbridge_suite)

The first version of this I believe origins from [Michael Jenkin](https://github.com/michaeljenkin), in the repo [unityros](https://github.com/michaeljenkin/unityros). He made a sample unity project showing turtlesim, with good instructions on how to use this project. All honor goes to him. I created this project because there was no repository containing the barebone library.

## Included messages
This repository does not contain every ROS message. If you need to add one, please make a pull request.

## Documentation
Documentation is in the code. I added some more in addition to what Michael Jenkin (original
author) did. The main file is ROSBridgeWebSocketConnection.cs, which sets up everything.

## Example usage
This is an example application where a ball is controlled. Basically, there are three important script types to notice. 

### **Setup the connection**
First, create a main script, **ROSInitializer.cs** responsible for initializing RosBridge. Attach this to a GameObject, say Main Camera:

``` cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using ROSBridgeLib;

public class ROSInitializer : MonoBehaviour
{
   public ROSBridgeWebSocketConnection ros = null;
    
  void Start() {
    // Where the rosbridge instance is running, could be localhost, or some external IP
    ros = new ROSBridgeWebSocketConnection ("ws://localhost", 9090);

    // Add subscribers and publishers (if any)
    ros.AddSubscriber (typeof(BallPoseSubscriber));
    ros.AddPublisher (typeof(BallTwistPublisher));

    // Fire up the subscriber(s) and publisher(s)
    ros.Connect ();
  }
  
  // Extremely important to disconnect from ROS. Otherwise packets continue to flow
  void OnApplicationQuit() {
    if(ros!=null) {
      ros.Disconnect ();
    }
  }
  // Update is called once per frame in Unity
  void Update () {
    ros.Render ();
  }
}

```
### **Subscriber**
Then, create a subscriber script, **BallPoseSubscriber.cs**, which will receive updates from a chosen ROS topic
``` cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

using ROSBridgeLib;
using SimpleJSON;
using ROSBridgeLib.geometry_msgs;


public class BallPoseSubscriber : MonoBehaviour
{
  static GameObject ball;
  
  // These two are important
  public new static string GetMessageTopic() {
    //Topic name is up to the user. It should return the full path to the topic. 
    //For eg, "/turtle1/cmd_vel" instead of "/cmd_vel".
    return "/pose_info";
  }

  public new static string GetMessageType() {
    //Same as the definition present in ROS documentation:
    return "geometry_msgs/Pose";
  }

  // Important function (I think.. Converts json to PoseMsg)
  public new static ROSBridgeMsg ParseMessage(JSONNode msg) {
    return new PoseMsg (msg);
  }

  // This function should fire on each received ros message
  public new static void CallBack(PoseMsg msg) {
    
    Debug.Log("Recieved Message : "+msg.ToYAMLString());
    // Update ball position, or whatever
    ball=GameObject.Find("ball");
    Vector3 ballPos=ball.transform.position;
    ballPos.x = msg.GetPosition().GetX();
    ballPos.y = msg.GetPosition().GetY();
    ballPos.z = msg.GetPosition().GetZ();
    //Changing ball's position to the updated position vector
    ball.transform.position=ballPos;
  }
}
```
Verify the subscriber by running:
```console
foo@bar:~$ rostopic pub -1 /pose_info geometry_msgs/Pose -- '[1.0,1.0,1.0]' '[1.0,1.0,1.0,1.0]' 
```
### **Publisher**
If you need to publish data to ROS, create a publisher like **BallTwistPublisher.cs**:
``` cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using SimpleJSON;
using ROSBridgeLib.geometry_msgs;

public class BallTwistPublisher : MonoBehaviour
{
    // The following three functions are important
  public static string GetMessageTopic() {
    //topic name is up to the user
    return "/twist_info";
  }

  public static string GetMessageType() {
      return "geometry_msgs/Twist";
  }

  public static string ToYAMLString(TwistMsg msg) {
    return msg.ToYAMLString();
  }

  public new static ROSBridgeMsg ParseMessage(JSONNode msg) {
    return new TwistMsg(msg);
  }    
}
```
Above script defines the publisher. In order to call this publisher attach one script,  **DataManager.cs**, to the gameobject ball.
```cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

using ROSBridgeLib;
using ROSBridgeLib.geometry_msgs;

public class DataManager : MonoBehaviour
{
    Rigidbody rb;
    GameObject rosObj;
    //Required for TwistMsg
    Vector3Msg linearVel;
    Vector3Msg angularVel;
    TwistMsg msg;
    
    // Start is called before the first frame update
    void Start()
    {
      //Since we attached ROSInitiazer to Main Camera:
        rosObj = GameObject.Find ("Main Camera");
        rb=GetComponent<Rigidbody>();
    }

    // Update is called once per frame
    void Update()
    {
      //dependant on the message defintion:
        linearVel=new Vector3Msg(
            rb.velocity.x,
            rb.velocity.y,
            rb.velocity.z
        );
        angularVel=new Vector3Msg(
            rb.angularVelocity.x,
            rb.angularVelocity.y,
            rb.angularVelocity.z
        );
        msg=new TwistMsg(linearVel,angularVel);
        rosobj.GetComponent<ROSInitialize>().ros.Publish(
            BallTwistPublisher.GetMessageTopic(),msg
        );
    }
}
```
verify the publisher script:
```console
foo@bar:~$ rostopic echo /twist_info
```
## License
Note: SimpleJSON is included here as a convenience. It has its own licensing requirements. See source code and unity store for details.

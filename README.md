This program and the accompanying materials are
made available under the terms of the Eclipse Public License v2.0 which accompanies
this distribution, and is available at https://www.eclipse.org/legal/epl-v20.html

SPDX-License-Identifier: EPL-2.0

Copyright Contributors to the Zowe Project.

# Sample React App

This branch acts as a tutorial, intended as a workshop session, which will teach you how to add App to App communication to a React App. 
The code here is the completed version, and serves as a reference. To complete this tutorial, you should either build off of a sandbox you have been using to complete the prior tutorials in sequential order, or, you can just [clone the previous tutorial branch here](https://github.com/zowe/react-sample-app/tree/lab/step-1-hello-world-complete) and work from that.

By the end of this tutorial you will:
1. Know how to receive and act on launch context given from other Apps or ZLUX
1. Know how to spawn a new App with a launch context you decide
1. Know how to message an open App to request it to do something

So, let's get started!

1. [Purpose of App to App Communication](#purpose-of-app-to-app-communication)
    1. [Communication is an Action](#communication-is-an-action)
1. [Setting Up for Rapid Development](#setting-up-for-rapid-development)
1. [Messaging Other Apps](#messaging-other-apps)
1. [Launching into Context](#launching-into-context)
1. [And More...](#and-more)

## Purpose of App to App Communication
When working with a computer, users tend to use multiple applications to accomplish some task, for example: checking a dashboard before digging into a detailed program or checking email before opening a bank statement in a browser. In many environments, the relationship between one program and another is loose and not well defined (you might open one program to learn of a situation which you solve by opening another and typing or pasting in content). Or perhaps a hyperlink is provided or attachment which opens up a program by using a lookup table of which program is the default for handling a certain file extension. 

The App framework attempts to solve this problem by creating a notion of structured messages that can be send from one App to another. An App has a context of what information is currently contained within it, which could be used to invoke an action on another App which would be best to deal with some information discovered in the first App. Well-structured messages facilitate knowing what App is "right" to deal with a situation, and explains in detail what that App should do. This way, rather than finding out that the attachment with the extension ".dat" was not meant for a text editor, but instead for an email client, one App may instead be able to invoke an action on an App which can handle opening of an email for the purpose of forwarding to others - a more specific task than can be explained with filename extensions.

### Communication is an Action

In order to manage communication from one App to another, a specific structure was needed. In the App framework, the unit of App-to-App communication is an **Action**. 
An Action always has a specific structure of data that is passed, to be filled in with the context at runtime, and a specific target that should receive the data. In addition, the Action is dispatched to the target in one of various modes: such as to target a specific existing instance of an App, any instance, or to create a new one. The **Action** may also be something less detailed than a message: It could be a request to minimize, maximize, close, launch, and more. Finally, all of this information is related to a unique ID and localization string such that it can be managed by the framework.

## Setting Up for Rapid Development

Before we get to implementing new features into this App, you should set up your environment to quickly build any changes you put in.
When building web content for ZLUX, Apps are packaged via Webpack, which can automatically scan for file changes on code to quickly repackage only what has changed.
To do this, you would simply run `npm run start`, but you may need to do a few tasks prior:

1. Open up a command prompt to `sample-react-app/webClient`
1. Set the environment variable MVD_DESKTOP_DIR to the location of `zlux-app-manager/virtual-desktop`. Such as `set MVD_DESKTOP_DIR=../../zlux-app-manager/virtual-desktop`. This is needed whenever building individual App web code due to the core configuration files being located in **virtual-desktop**
1. Execute `npm install`. This installs all the dependencies in the **package.json** within the directory.
1. Execute `npm run start`

## Messaging Other Apps

OK, now we can get to the code. Your App should currently have some elements on the left side of its window for messaging Apps, added from the previous tutorials, but so far the logic for it has been stubbed out.
Let's add that logic in just two places.

Open up **sample-react-app/webClient/src/App.tsx** and change the method **sendAppRequest** to the following:

```typescript
  sendAppRequest() {
    var requestText = this.state.parameters;
    var parameters = null;
    /*Parameters for Actions could be a number, string, or object. The actual event context of an Action that an App recieves will be an object with attributes filled in via these parameters*/
    try {
      if (requestText !== undefined && requestText.trim() !== "") {
        parameters = JSON.parse(requestText);
      }
    } catch (e) {
      //requestText was not JSON
    }
    
    let appId = this.state.appId;  
    if (appId) {
      let message = '';
      /* With ZLUX, there's a global called ZoweZLUX which holds useful tools.
         PluginManager can be used to find what Plugins (Apps are a type of Plugin) are part of the current ZLUX instance.
         Once you know that the App you want is present, you can execute Actions on it by using the Dispatcher.
      */              
      let dispatcher = ZoweZLUX.dispatcher;
      let pluginManager = ZoweZLUX.pluginManager;
      let plugin = pluginManager.getPlugin(appId);
      if (plugin) {
        let type;
        type = dispatcher.constants.ActionType[this.state.actionType];
        let mode;
        mode = dispatcher.constants.ActionTargetMode[this.state.appTarget];
        
        if (type != undefined && mode != undefined) {
          let actionTitle = 'Launch app from sample app';
          let actionID = 'org.zowe.zlux.sample.launch';
          let argumentFormatter = {data: {op:'deref',source:'event',path:['data']}};
          /*Actions can be made ahead of time, stored and registered at startup, but for example purposes we are making one on-the-fly.
            Actions are also typically associated with Recognizers, which execute an Action when a certain pattern is seen in the running App.
          */
          let action = dispatcher.makeAction(actionID, actionTitle, mode,type,appId,argumentFormatter);
          let argumentData = {'data':(parameters ? parameters : requestText)};
          this.log.info((message = 'App request succeeded'));        
          this.setState({status: message});
          /*Just because the Action is invoked does not mean the target App will accept it. We've made an Action on the fly,
            So the data could be in any shape under the "data" attribute and it is up to the target App to take action or ignore this request*/
          dispatcher.invokeAction(action,argumentData);
        } else {
          this.log.warn((message = 'Invalid target mode or action type specified'));        
        }
      } else {
        this.log.warn((message = 'Could not find App with ID provided'));
      }
      this.setState({status: message});
    }
  }
```

Then, we'll edit the constructor of App, and add a new method, **getDefaultState** to set some defaults for some of the variables referenced. These should now look like:

```typescript
  constructor(props){
    super(props);
    this.log = this.props.resources.logger;
    this.state = this.getDefaultState();
  };

  private getDefaultState() {
    return {
      actionType: "Launch",
      appTarget: "PluginCreate",
      parameters: 
      `{"type":"connect",
        "connectionSettings":{
          "host":"localhost",
          "port":23,
          "deviceType":5,
          "alternateHeight":60,
          "alternateWidth":132,
          "oiaEnabled": true,
          "security": {
            "type":0
          }
        }}`,
      appId: "com.rs.mvd.tn3270",
      status: "Status will appear here.",
      helloText: "",
      helloResponse: "",
      destination: ZoweZLUX.uriBroker.pluginRESTUri(this.props.resources.pluginDefinition.getBasePlugin(), 'hello',"")
    };
  }
```

What you see above is that we utilize the **Dispatcher**, a core component of ZLUX, to request operations on other Apps. However, before we do that, we make use of the **PluginManager** to verify that the App
The user requested exists in the environment. On the dispatcher, you can make a new **Action** or invoke a previously made one. Zowe can load Actions from system defaults and user preferences at startup, so it's
not necessary to make them on the fly as we do here, but it helps demonstrate how they work.

The Action that we make depends on user input, but you can see that an **Action** is composed of a unique ID (for future reference), a title (to internationalize, perhaps), a mode of operation, a way to target an App, which App to target, and a template of arguments to provide. The template is important in that it allows for rich information to be formatted in a way that the receiving App can handle. It's a formality so that both
the caller and the callee will be able to send a message that's understood. In the case of this sample App, however, we just wrap everything the user has input into a top-level attribute, `data`. This is not always
something a receiving App would expect, so it's just for limited demonstration use.

At this point, you should be able to run the App and have it be able to tell another App what to do. The default example provided is a message that the TN3270 terminal App understands. Simply, try changing the hostname and port to something other than localhost:23 and see that the terminal can be opened to that destination!

## Launching into Context

One of the operations the user can do with the App right now is to tell another App to open and do something on opening. But, how is that done on the other App? We can find out by adding such support to this App.

This part's short - just edit the constructor and add two convenience methods:

```typescript
  constructor(props){
    super(props);
    this.log = this.props.resources.logger;
    let metadata = this.props.resources.launchMetadata;
    if (metadata != null && metadata.data != null && metadata.data.type != null) {
      this.handleLaunchOrMessageObject(metadata.data);
    } else {
      this.state = this.getDefaultState();
    }

  };
  
  updateOrInitState(stateUpdate: any): void {
    if (!this.state) {
      this.state = Object.assign(this.getDefaultState(), stateUpdate);
    }
    else {
      this.setState(stateUpdate);
    }
  }

  handleLaunchOrMessageObject(data: any) {
    switch (data.type) {
    case 'setAppRequest':
      let actionType = data.actionType;
      let msg:string;
      if (actionType == 'Launch' || actionType == 'Message') {
        let mode = data.targetMode;
        if (mode == 'PluginCreate' || mode == 'PluginFindAnyOrCreate') {
          this.updateOrInitState({actionType: actionType,
                                  appTarget: mode,
                                  appId: data.targetAppId,
                                  parameters: data.requestText});
        } else {
          msg = `Invalid target mode given (${mode})`;
          this.log.warn(msg);
          this.updateOrInitState({status: msg});
        }
      } else {
        msg = `Invalid action type given (${actionType})`;
        this.log.warn(msg);
        this.updateOrInitState({status: msg});
      }
      break;
    default:
      this.log.warn(`Unknown command (${data.type}) given in launch metadata.`);
    }
  }  
```

What you'll see here is that listening and acting upon a request you get on opening a new instance of your App is really simple: there's just one object, `this.props.resources.launchMetadata`, which either exists or doesn't depending on if the instance was launched with a request to do something special.

Now, within your App, you get to determine if the object given is something supported and able to do at that time. The object itself could have anything within, so the app must check it to determine what actions to take.

At this time, you've got the basics of App-to-App communication down! Your App can send and listen on requests, so you've completed this tutorial.


## And More...

This tutorial only covers a few ways in which you can act on other Apps within Zowe. For more information on advanced topics, such as how to load Actions from files rather than hardcode, or how to invoke an Action dependent upon a condition being met, [check the wiki these topics and more](https://github.com/zowe/zlux/wiki/App-to-App-Communication).



This program and the accompanying materials are
made available under the terms of the Eclipse Public License v2.0 which accompanies
this distribution, and is available at https://www.eclipse.org/legal/epl-v20.html

SPDX-License-Identifier: EPL-2.0

Copyright Contributors to the Zowe Project.

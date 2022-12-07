# Moving to Publisher Connector
Moving to Publisher Interface from using `editorObject` directly can be a one day event. This is especially true if you are using CHILI publish Online. However, not all integrations are the same. This documentation will help you move successfully and point out common areas of difficulty.

Before going any further, you will want to read [Differences With editorObject](https://github.com/chili-publish/publisher-interface/wiki/Differences-With-editorObject) to understand what are the major differences between `PublisherInterface` and `editorObject`. When you have done that, then continue with this documentation.

When moving to Publisher Interface, the areas of concern are:
* [Handling `PublisherInterface` API returns Promises](#handling-publisherinterface-api-returns-promises)
* [Dealing with `PublisherInterface` API name changes](#dealing-with-publisherinterface-api-name-changes)
* [Relying on auto-listened events](#relying-on-auto-listened-events) *DocumentFullyLoaded*, *SelectedPageChanged*, or *DocumentSaved* events
* [Using the `window.OnEditorEvent()` function](#using-the-windowoneditorevent-function)
* [Relying on the `window.OnEditor()` function](#relying-on-the-windowoneditor-function)
* [Relying on the `window.OnGetApiKey()` function](#relying-on-the-windowongetapikey-function)

<br/>

Note: If you are not using CHILI publish Online, but an older hosted version of CHILI, there is some special steps you must implement. Please also read [Self Hosted](#self-hosted).

<br/>

## Handling `PublisherInterface` API returns Promises
This is by far going to be the most difficult change for some integrations.

As you read in [Differences With editorObject](https://github.com/chili-publish/publisher-interface/wiki/Differences-With-editorObject), the `PublisherInterface` returns a Promise instead of the value directly.

✏️ The editorObject way returns a direct value:
```javascript
function logDocName() {
  const documentId = editorObject.GetObject("document.id");
  console.log(documentId);
}
```

💻 The publisher way returns a Promise which we can handle with a callbacks
```javascript
function logDocName() {
  publisher.getObject("document.id").then(
      documentId => console.log(documentId)
  );
}
```

or wrapped in an `async` function:
```javascript
async function logDocName() {
  const documentId = await publisher.getObject("document.id");
  console.log(documentId);
}
```

<br/>

Those are your two options to handling a Promise:
1. callbacks
2. `await` inside an `async` function

The await/async pattern is going to provide a similar experience to a synchronous `editorObject`-like workflow.

However, the problem is that async/await pattern directly influences any enclosing function or dependant function.

So say for example, your integration has a script that requires this order:

```javascript
function printNames(editorObject, print) {
  if (checkNameVarsExist(editorObject)) {
    const names = getName(editorObject);
    print(names);
  }
  else {
    print({first:"?", last: "?"})
  }
}

function checkNameVarsExist(editorObject) {
  return (editorObject.GetObject("document.variables[First Name]") != null) && (editorObject.GetObject("document.variables[Last Name]") != null);
}

function getNames(editorObject) {
  return {
    first: editorObject.GetObject("document.variables[First Name].value"),
    last: editorObject.GetObject("document.variables[Last Name].value")
  }
}
```

`printNames()` will be called by the application to get the values of the first and last name. We pass in the `editorObject` and some function called `print` that will handle the app logic of these variable values.

If we switch this script to using a `PublisherInterface` instance, and we want to use the async/await pattern to keep things logically the same. We would update our code to look like so:

```javascript
function printNames(editorObject, print) {
  if (checkNameVarsExist(editorObject)) {
    const names = getName(editorObject);
    print(names);
  }
  else {
    print({first:"?", last: "?"})
  }
}

async function checkNameVarsExist(editorObject) {
  return (await editorObject.GetObject("document.variables[First Name]") != null) && (await editorObject.GetObject("document.variables[Last Name]") != null);
}

async function getNames(editorObject) {
  return {
    first: await editorObject.GetObject("document.variables[First Name].value"),
    last: await editorObject.GetObject("document.variables[Last Name].value")
  }
}
```

<br/>

This is great, but when you run this code you will find `printName` does not behave as expected. If you run `checkNameVarsExist` or `getNames` in isolation, they work, but altogether `printName` does not.

The reason is that `printName` is not an `async` function. So the first if statement is comparing `Promise<boolean> == true` which evaluates to be false.

The inner `async` functions influence the outer dependant `printName` function. So that means `printName` should follow the async/await pattern too.


```javascript
async function printNames(editorObject, print) {
  if (await checkNameVarsExist(editorObject)) {
    const names = await getName(editorObject);
    print(names);
  }
  else {
    print({first:"?", last: "?"})
  }
}
```

<br/>

This was a very simple example, but you can see how one `async` function will basically make all functions that depend on that one also be `async`. Since most likely your integration script is written in non-async functions, the Publisher Interface API will require a rethinking from the bottom up.

For some integrations, it will just be adding `async` and `await` keywords all the way to the top. For others, it may mean that certain logic will need to be redone with asynchronous in mind.

Either way, this is the hardest part for any integration. So if you can get this resolved, everything else is easy.

<br/>

## Dealing with `PublisherInterface` API name changes
As you read in [Differences With editorObject](https://github.com/chili-publish/publisher-interface/wiki/Differences-With-editorObject), the API for the `PublisherInterface` class has changed to be camelCase.

✏️ The editorObject way:
```javascript
editorObject.ExecuteFunction("document", "Save");
```

💻 The publisher way:
```javascript
await publisher.executeFunction("document", "Save");
```

<br/>

There is a convenience getter on the `PublisherInterface` class named [editorObject](PublisherInterface.md#editorobject) which acts as an alias that can help current implementations switch to Publisher Interface.

Using the getter we could rewrite our publisher example as: 
```javascript
await publisher.editorObject.ExecuteFunction("document", "Save");
```

If your integration has lots of method calls to the `editorObject` then this getter can be used to save some modification time. You can even capture this in a variable called *editorObject* or *editor* to then pass onto your code.

```javascript
const editorObject = publisher.editorObject
await editorObject.ExecuteFunction("document", "Save");
```

There is one danger in using this convenience getter. That danger is you forget about the `async` keyword. This convenience getter does not magically make Promises into original values. The way Publisher Interface uses a message system means that getting rid of callbacks or async/await is not possible.

So when handling the name changes you have two choices:
1. Change the names of all your functions to make the Publisher Interface API.
2. Use the convenience getter: [editorObject](PublisherInterface.md#editorobject), but not to forget about async/await.

## Relying on auto-listened events
With the subdomain trick, there we three events that got sent to `window.OnEditorEvent()` without having to use `editorObject.addListener()`. These event were: *DocumentFullyLoaded*, *SelectedPageChanged*, and *DocumentSaved*.

With Publisher Interface, these events will not be sent to `window.OnEditorEvent` by default.

However, you can emulate this behavior by adding in an optional parameter when building an instance of `PublisherInterface`.

```javascript
await PublisherInterface.build(iframe, {events:["DocumentFullyLoaded", "SelectedPageChanged", "DocumentSaved"]})
```

This will cause `PublisherInterface` to auto-listened to all three of these events.

<br/>

FYI, you can add any events you want to this list, and they too will be auto-listened

<br/>

## Using the `window.OnEditorEvent()` function
Good news, nothing changes besides the fact that `addListener()` returns a Promise.

So, this works as expected:
```javascript
await publisher.addListener("FrameMovedFinished");
```

<br/>

## Relying on the `window.OnEditor()` function
If you used OnEditor method and passed in a callback function like so:

✏️ The editorObject way:
```javascript
// Function called on onload event of iframe
function iFrameLoaded()
{
    iframe = document.getElementById("chili-iframe");
    iframe.contentWindow.GetEditor(myCallback);
}

function myCallback(editorObject)
{
  // Do something with editorObject
}
```
This will no longer work with the Publisher Interface.

The best suggestion is to rework your logic around the waiting for the `build()` method. However, if you really must, can create similar behavior with `build()`.

⚠ Very important difference, unlike `GetEditor()`, `build()` must be called before the iframe `onload` event.

```javascript
// This must be done before iframe onload event
function beforeIFrameLoaded() {
  iframe = document.getElementById("chili-iframe");
  PublisherInterface.build(iframe).then(myCallback);
}

function myCallback(publisher)
{
  // Do something with publisher
}
```

Again, it might make more sense to use async/await

```javascript
// This must be done before iframe onload event
function beforeIFrameLoaded() {
  iframe = document.getElementById("chili-iframe");
  const publisher = await PublisherInterface.build(iframe);
  myCallback(publisher);
}

function myCallback(publisher)
{
  // Do something with publisher
}
```

<br/>

Two notes:
* Nobody should have used `window.EditorLoaded()`, but the same logic above would apply to that function.
* The iframe would need to have a `src`. So the order is set `src` then call `beforeIFrameLoaded()`.

<br/>

## Relying on the `window.OnGetApiKey()` function
There is no replacement or workaround for utilizing this function.

The function `OnGetApiKey()` was called by the Publisher editor on load if the API key was missing in the URL of the editor. This function was expected to return an API key.

The inspiration for `OnGetApiKey()` was to hide the API key through obscurity. However, I would argue that this design was ill fated, and thus not included in Publisher Interface.

The `OnGetApiKey()` function really did nothing to hide the API key, as the key was easily visible in the network tab as well as in an integrations JavaScript. A better option would be practice safe API key creation, and to always generate a key with the minimum number of permissions.

<br/>

## Self Hosted
If you are on an older self-hosted CHILI publish, then you will need two things:
* Be on a version of CHILI Publish greater than 6.5.5
* Edit editor_html.aspx

<br/>

### Latest Version
Due to some code modifications required within the application to support the PublisherInterface, if you attempt to use this package on a self-hosted version of CHILI older than version 6.5.5, then everything will work *except* for the event system via `addListener`.

In order to avoid this, please make sure your CHILI version is greater than 6.5.5. You are able to find the latest downloads [here](https://mysupport.chili-publish.com/hc/en-us/articles/360021250259-Latest-downloads). 

We also have an article on our support website with information about updating your CHILI installation located [here](https://mysupport.chili-publish.com/hc/en-us/articles/4405609364370-Safe-Update)

<br/>

### Edit
Find your editor_html.aspx file, which is typically located at _C:\CHILI_Publisher\app\editor_html.aspx_, but you can always look up the path in your IIS settings.

Once you find the file, open it in an editor, and go down to the very end of `</body>`.

Before the close tag `</body>`, add this script tag:
```html
<script type="text/javascript" src="https://unpkg.com/@chili-publish/publisher-interface@latest/dist/chiliInternalWrapper.min.js"></script>
```

Therefore, your `editor_html.aspx` should end like this:
```html
    </div>
	<script type="text/javascript" src="https://unpkg.com/@chili-publish/publisher-interface@latest/dist/chiliInternalWrapper.min.js"></script>
</body>
</html>
```

If you do not want to or cannot use unpkg, you can build the script yourself. See [onServer README](https://github.com/chili-publish/publisher-interface/blob/main/onServer/README.md) for more information.

<br/>

There are two things to take note:
* You will need to edit the `editor_html.aspx` after every update. We may potentially include a script that will make this easier.
* You will know it is working because you can find unpkg in the `Sources` of all Chromium based browser.
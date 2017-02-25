Introducing xcore
-----------------


----------


**Please be aware there are no binaries available yet**
**I am still working on the project, but feedback and wishes are welcome**


----------


###What is xcore###
xcore is a [Node.js](https://nodejs.org)® plugin enabling the use of .NET [netstandard](https://github.com/dotnet/standard/) libraries with javascript.

 - Node.js® is a well-known runtime environment https://nodejs.org for javascript
 - netstandard is a new specification by Microsoft representing a set of .NET API which can be used across many .NET implementations such as .NET Framework and .NET Core.
	 - https://github.com/dotnet/standard

Basically xcore allows any Node.js script to use .NET code.
xcore is entirely authored by Raffaele Rialdi  -  @raffaeler 

###Show me the code ...###

Load and initialize xcore

    var xcore = require('bindings')('xcore');
    xcore.initialize(__dirname + "\\Sample",
        "SampleLibrary.dll", "SampleLibrary.Initializer");
        
Load the .net metadata. This operation is done for just the entry-point class of a graph.

    xcore.loadClass("SampleLibrary.OrderSample.    OrderManager, SampleLibrary");
Create an instance of the .NET class

    var om = new xcore.OrderManager("raf");
Call methods

    console.log(om.Add("Hello, ", "world"));
Walk the graph. Metadata for the Order class is loaded automatically

    console.log(om.SelectedOrder.Name);
Asynchronous calls 

    om.AddAsync(2, 3, function(res){
    console.log(res);
});

Subscribing events

    var cookie = om.addEventHandler("OrderReady", function(sender, args){
        console.log("sender: " + sender.SelectedOrderName, " args:" + args.Name);
        });

Unsubcscribe events
	
    om.removeEventHandler("OrderReady", cookie);

Much more is supported!

###Features already implemented###

 - Creating .NET objects – calling constructors
	 - Constructors and methods supports overloads
 - Invoking methods, including asynchronous Task<T> methods
	 - They will be executed in the context of the calling thread (the main JS thread)
	 - If asynchronous, the callback will be enqueued back to the main thread
 - Using properties and indexers
	 - Normally used to access the state of the created instance
 - Subscribing events (support for events invoked on secondary threads)
	 - The code can refer to the "cookie" (see example) or directly to the subscribing function
 - Automatic upcasting and downcasting
	 - In the event sample, the "sender" is an object, but is downcasted to the runtime object

###How does it work?###

N/A

###xcore on the C++ side###

- C++ layer is invisible to both JS and .NET developers
- Host the CoreCLR and load the required infrastructure
- Creates proxies "on demand" for each .NET object you want to access
- Dynamically map all the required constructors/methods/properties/events
- .NET metadata is used to build a complete map
- Each call from Javascript is mapped by xcore to the .NET instance/member
- If a method returns Task or Task<T>, a different (asynchronous) mapping occurs
- Interop occurs to talk "Reverse PInvoke" to .NET code

### xcore on the .NET side ###
- .NET layer is invisible to both JS and .NET developers
- Receives the requests from C++ code for each constructor/method/property/event
- Each time a method is accessed, xcore generates code doing the interop
- After the first call, each access to a member is an optimized fast delegate
- Execute the .NET user code
- Maintains a cache of the marshaled instances (tied to the JS garbage collector)
- Calls back the C++ code with the responses (PInvoke)
- A special treatment is done for asynchronous calls

###Asynchronous code - promises  -  async / await###
- V8 engine uses libuv to manage the threading story
- Javascript / Node.js / V8 must execute code in the one and only specific thread
- The xcore plugin leverages the V8 engine to "fix" the calls from different threads
- Constructors, properties and methods normally are all executed on the JS thread
- Methods returning tasks and events are executed asynchrously and calls are pushed back to custom queues fixing the thread mismatch
- Two arguments (functions) are added at the end of the signature of the JS proxy
	- onSuccess and onError will be called, if provided, as soon as the results are available

####Events###
- Javascript has no direct support for events
- The xcore plugin add two members to all the proxied objects:

    addEventHandler("OrderReady", function(sender, args) { … });
    removeEventHandler("OrderReady", function(sender, args) { … });

- The name of the function can be used instead of the function implementation
- To ease unsubscription, it can also be used a different syntax:

    var cookie = addEventHandler("OrderReady", function(sender, args) { … });
    removeEventHandler("OrderReady", cookie);

- The program execution will not end until all the subscriptions are correctly unsubscribed.
###Performance profile###
- There are still many improvements that can be done
- The add use-case is not realistic and is used to measure the infrastructure
	- Four marshaling operations will be always needed (JS -> C++ -> .NET -> C++ -> JS)

####Initialization code####

    var om = new xcore.OrderManager();
    
    om.Add(2, 2);		// jitting
    LocalAdd(2,2);		// jitting
    xcore.add(2, 2);	// jitting
    var loop = 1000000;	// 1 Million
    
    function LocalAdd(x, y)
        { return x+y; }

####Javascript only####

    console.time("js");
    for(var i=0; i<loop; i++)
    {
        LocalAdd(i, i);
    }
    console.timeEnd("js");

####C++ static function, optimized ad-hoc Add####

    console.time("c++");
    for(var i=0; i<loop; i++)
    {
        xcore.add(i, i);
    }
    console.timeEnd("c++");
####.NET Add performed via xcore####

    console.time("net");
    for(var i=0; i<loop; i++)
    {
        om.NetAdd(i, i);
    }
    console.timeEnd("net");

####Results for 1 million calls####

    	  js:	 6.156 ms
    	 c++:	56.352 ms
    	.net: 1294.254 ms
###Performance considerations###
- The "Add" case is the worst possible case
	- JS code is compiled to machine code
	- C++ code is optimized but we pay the cost of V8 engine (parameters marshaling)
	- .NET code pays the V8 overhead + the .NET marshaling
		- On the .NET side the dynamically created code is fast (no reflection involved)
		- We still have to find the right instance to call the NetAdd method on
-	More can be done by branching and modifying the CLR itself
	-	Certainly not coming soon (if ever)
###Use-cases###
1. Node.js applications
2. UI created with the [Electron framework](http://electron.atom.io)
3. Using JS to script Windows Powershell
4. [Nativescript](https://www.nativescript.org/) Mobile applications
5. Anything else based on V8

###Possible future steps###
- Port the whole thing on Linux and mobile platforms
- Create Typescript mappings automatically
- Support straight delegates to support reactive extensions like paradigm (easy to do, the event infrastructure already provides the needed infrastructure)
- Support "projections" (renaming members to adapt the javascript naming conventions)
- The necessary infrastructure is already there, but still not exposed
- Support for fields (easy to do, not sure if it is really needed)
- Support for javascript iterators (for – of)
- Specialized plugin to use Javascript to script Windows Powershell
- Other dark secret plans :)

###Source code###
I have no current plans to release the source code for xcode.
###Release date###
There are still issues I have to fix before making it a stable thing that can be used in production. Basically I have to carefully check for memory leaks problems and fixing lifecycle issues.
For this reason I did not release the plugin at this moment.
Follow this repo or myself on twitter [@raffaeler](https://twitter.com/raffaeler) for any news.
###Suggestions and requests###
Please open an issue on this GitHub repo to ask, discuss and propose anything about xcore.
Even better, post the javascript + .NET essential snippets you'd like to use but not sure it would work.


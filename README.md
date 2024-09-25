# Adding a New Module to CodeProject.AI Server

In this article we explore how CodeProject.AI Server really works, and how simple it is to add your own functionality to CodeProject.AI server quickly and easily, leaving you to focus on your business, not your MLOps

![](https://raw.githubusercontent.com/ChrisMaunder/Adding-a-New-Module-to-CodeProject-AI-Server/master/docs/assets/ashkan-forouzani-m0l9NBCivuk-unsplash.jpg)

## Introduction

[Adding AI capabilities to an app](https://www.codeproject.com/Articles/5330374/5-ways-to-add-Artificial-Intelligence-to-an-existi) is reasonably straight forward if you're happy to follow the twisty turny maze that is the endless list of libraries, tools, interpreters, package managers and all the other fun stuff that sometimes makes coding about as fun as doing the dishes.

We built [CodeProject.AI Server](https://www.codeproject.com/Articles/5322557/CodeProject-AI-Server-AI-the-easy-way) to hide all the annoying things from developers and leave them with a simple AI package that Does Stuff and is easy to use with an existing application.

If CodeProject.AI Server doesn't do what you need, then it's easy to add modules that will fill in the gap.

## Aggregating, not adding

We say "add", but "aggregating" is more accurate. There is a *ton* of amazing AI projects out there being actively developed and improved and we want to allow developers to take these existing, evolving AI modules or applications and drop them into the [CodeProject.AI](http://codeproject.ai/) ecosystem with as little fuss as possible. This could mean dropping in a console application, a Python module, or a .NET project.

For development, all you need to do is

1. **Create an install script** (usually short) to setup the pre-requisites (download models, install necessary libraries, create some folders if needed)
2. **Write an adapter** that handles communication between the module and the [CodeProject.AI](http://codeproject.ai/) server,
3. **Provide a `modulesettings.json` file** that describes the module and provides instruction to [CodeProject.AI](http://codeproject.ai/) on how to launch the module.

## The CodeProject.AI Server Architecture in under 30 Seconds

CodeProject.AI Server is an HTTP based REST API server. It's basically just a webserver to which your application sends requests. Those requests are placed on a queue, and the analysis services (aka The Modules) pick requests off the queues they know how to service. Each request is then processed (an AI operation is performed based on the request) and the results are sent back to the API server, which in turn sends it back to the application that made the initial call.

Think of CodeProject.AI Server like a database server or any other service you have running in the background: it runs as a service or daemon, you send it commands and it responds with results. You don't sweat the details of how it goes about its business, you just focus on your application's core business.

The CodeProject.AI Server API Server runs independently of the calling application.

As an example, suppose we had three analysis modules, Face recognition using Python 3.7, Object Detection using .NET, and Text Analysis using Python 3.10:

![](https://raw.githubusercontent.com/ChrisMaunder/Adding-a-New-Module-to-CodeProject-AI-Server/master/docs/assets/CodeProjectAI-arch.png)

1. An application sends a request to the API server
2. The API server places the request on the appropriate queue
3. The backend modules poll the queue they are interested in, grab a request and process it
4. The backend module then sends the result back to the API server
5. The API Server then sends the result back to the calling application

Each backend module can be on any tech stack and run under whatever runtime is installed - Python, .NET, Go, C++ - whatever can run on the host system. It's up to the module's installer to ensure the host system has whatever parts are needed installed, but CodeProject.AI Server provides lots of help for this.

## Adding a New Module to CodeProject.AI Server

There are three tasks in adding a new module:

1. Creating a settings file that instructions the server and the installer what your module needs
2. Ensuring any prerequisites such as models, libraries or interpreters are installed correctly by way of an install script
3. Writing the actual module and wiring it up to CodeProject.AI Server

### The settings file

This is a simple JSON file that provides general information such as name, version and description of the module., as well as the runtime used, version compatibility information, GPU usage guidance, and routing info. The routing info allows the server to listen on a given API route and pass those request to your module.

### The install script

The install script (and post-install script) provides an opportunity to download models, install runtimes, build and install libraries, and perform any tasks needed to ensure the module can run. You will need a Windows BAT file for the Windows installer, and a bash file for the Linux / macOS installers.

The latest version of CodeProject.AI server does not need the install script to install Python or .NET - these are taken care of by the core setup system based on your settings file. If the setup system sees the module is using .NET or Python it will setup these environments, and for Python it will also look for and install the packages in the requirements file into a virtual environment. 

Other runtimes or supporting services should be added by the install script.

#### Writing a Module

Writing a module is the fun part. In fact, you often don't have to write a new module: there are hundreds of excellent Open Source, self-contained AI projects that would make excellent modules. All you need to do is ensure the module can run in the installed environment (taken care of in the pre-requisites step), that any models it needs are downloaded and in the right place (again, should already be done), and that the module can communicate with the CodeProject.AI Server.

We will be providing a simple SDK for many languages that will help allow you to write a shim that will fit between the module and CodeProject.AI Server and take care of communication.

## Let's Add a Module

We're going to add the [rembg](https://github.com/danielgatis/rembg) module. This is a simple but fun AI module that takes any photo containing a subject and removes the background from the image. It runs under Python 3.9 or above.

### Setup (the installer)

The `rembg` module comprises the following:

1. The Python code
2. The Python 3.9 interpreter
3. Some Python packages
4. The AI models

To ensure these are all in place within the development environment, we need to modify the setup scripts in */Installers/Dev*.

#### For Windows (install.bat)

```cpp
:: Background Remover :::::::::::::::::::::::::::::::::::::::::::::::::::::::::

call "%sdkScriptsDirPath%\utils.bat" GetFromServer "rembg-models.zip" "models" "Downloading Background Remover models..."

```cpp
This one doesn't get much simpler: it simply downloads the models from the CodeProject S3 bucket and moves the files into place. The `GetFromServer `function is included in the SDK scripts and takes a filename, a folder name to extract the file into (relative to the current module) and a message

#### For Linux or macOS

The script is essentially the same as the Windows version:

```cpp
# Background Remover :::::::::::::::::::::::::::::::::::::::::::::::::::::::::

getFromServer "rembg-models.zip" "models" "Downloading Background Remover models..."
```

### The Module's API

We should start with how we'll call the module. It can be whatever we like, so let's choose the route */v1/image/removebackground*. We'll pass in an image and a boolean `use_alphamatting` which tells the code whether or not to use alpha matting (better for fuzzy edges).

The return package will include a single item `imageBase64`, which contains the `base64` encoded version of the image with background removed.

### The Module's Source Code

First, we create a folder under the *modules* directory and copy over the code for the module. In this case, we'll store the code in */src/modules/BackgroundRemover*. For convenience, we'll create a Python project for those using Visual Studio (working in VS Code is just as simple).

The `rembg` module has one main method we need to call, named `remove`. We need to be able to get the data from a client's request to this method, and then pass the results of this method back to the client. For this, we'll use the *src/SDK/Python/module\_runner.py* module to help.

We'll also create an adapter (which we'll call *rembg\_adapter.py*) for `rembg` that connects the `rembg remove` method with our *codeprojectAI.py* helper module.

```cpp
# Import our general libraries
import sys
import time

# Import the CodeProject.AI SDK. This will add to the PATH var for
# future imports
sys.path.append("../../SDK/Python")
from request_data import RequestData
from module_runner import ModuleRunner
from common import JSON

# Import the method of the module we're wrapping
from PIL import Image

# Import the method of the module we're wrapping
from rembg.bg import remove

class rembg_adapter(ModuleRunner):

    def initialise(self) -> None:   
        """ Initialises the module """
        pass

    def process(self, data: RequestData) -> JSON:
        """ Processes a request from the client and returns the results"""
        try:
            img: Image             = data.get_image(0)
            use_alphamatting: bool = data.get_value("use_alphamatting", "false") == "true"

            # Make the call to the AI code we're wrapping, and time it
            start_time = time.perf_counter()
            (processed_img, inferenceTime) = remove(img, use_alphamatting)
            processMs = int((time.perf_counter() - start_time) * 1000)

            return { 
                "success":      True, 
                "imageBase64":  RequestData.encode_image(processed_img),
                "processMs" :   processMs,
                "inferenceMs" : inferenceTime
            }

        except Exception as ex:
            self.report_error(ex, __file__)
            return {"success": False, "error": "unable to process the image"}

    def shutdown(self) -> None:
        pass

if __name__ == "__main__":
    rembg_adapter().start_loop()
```

This is the only code we've added. The `rembg` module has been copied and pasted as-is, and we're reusing the *ModuleRunner* worker class. Nothing else (code-wise) needs to be added.

### The modulesettings.json File

This file, in the *BackgroundRemover* folder, instructs the API server in how to launch our new analysis service.

```cpp
{
  "Modules": {

   "BackgroundRemoval": {

      "Name": "Background Remover",
      "Version": "1.6.2",

      // Publishing info
      "Description": "Automatically removes the background from a picture", 
      "Platforms": [ "windows", /*"linux",*/ "linux-arm64", "macos", "macos-arm64" ], // issues with numpy on linux
      "License": "SSPL",
      "LicenseUrl": "https://www.mongodb.com/licensing/server-side-public-license",

      // Launch instructions
      "AutoStart": false,
      "FilePath": "rembg_adapter.py",
      "Runtime": "python3.9",
      "RuntimeLocation": "Local",       // Can be Local or Shared

      "EnvironmentVariables": {
        "U2NET_HOME": "%CURRENT_MODULE_PATH%/models" // where to store the models
      },

      "RouteMaps": [
          // ... (explained below)
      ]
    }
  }
}
```

The `EnvironmentVariables` section defines key/value pairs that will be used to set environment variables that may be required by the module. In this case, the path to the AI model files. This is a value specific to, and defined by, the rembg module.

`CURRENT_MODULES_PATH` is a macro that will expand to the location of the directory containing the modules. In this case, */src/AnalysisLayer*.

The `FilePath` is the path to the file to be executed, relative to the current module's directory.\

`AutoStart` sets whether or not this module will be launched at when the server starts up.

Runtime defines what runtime will launch the file. We currently support `dotnet` (.NET), `python3.6` to `Python 3.12`, or just `python` if you will use the system default version of Python. If omitted, the CodeProject.AI Server will attempt to guess based on the `FilePath`.

The `Platforms `array contains an entry for each platform on which the service can run. Currently, Windows, Linux, macOS and Docker are supported.

The file also defines the API routes for the module under the `RouteMaps` section

```cpp
{
  "Modules": {
    "ModulesConfig": {
      "BackgroundRemoval": {
         "Name": "Background Removal",
         "Description": "Removes backgrounds from images.",

         ...

         "RouteMaps": [
           {
             "Path": "image/removebackground",
             "Command": "removebackground",
             "Description": "Removes the background from images.",
             "Inputs": [ ... ],
             "Outputs": [...]
           }
         ]
       }
     }
   }
}
```

Path is the API path, in this case *localhost:5000/v1/image/removebackground*. Remember that this was what we chose (arbitrarily) as our API. It can be anything as long as it isn't currently in use.

`Command` is the method in the API controller that will be called, in this case `removebackground`. `Queue` is the name of the queue in the API server that will manage the request for this service.

In the adapter Python module we wrote, we had the code:

```cpp
QUEUE_NAME = "removebackground_queue"
```

`Queue`, in the route map, should match this name.

`Description`, `Inputs `and `Outputs `are purely documentation at this stage.

### The Client that will Call Our New Module

A simple JavaScript test harness can be used to demonstrate the new module.

```cpp
  // Assume we have a HTML INPUT type=file control with ID=fileChooser
  var formData = new FormData();
  formData.append('image', fileChooser.files[0]);
  formData.append("use_alphamatting", 'false');

  var url = 'http://localhost:5000/v1/image/removebackground';

  fetch(url, { method: "POST", body: formData})
        .then(response => {
            if (response.ok) {
                response.json().then(data => {
                    // img is an IMG tag that will display the result
                    img.src = "data:image/png;base64," + data.imageBase64;
                })
            }
        })
```

The project contains a file *test.html* that implements this, providing the UI to collect the information and display the results.

## Install and Test

At this point, we have a module, an install script and a test client. Let's give it a run.

1. Ensure you have the latest [CodeProject.AI Server repo](https://github.com/codeproject/CodeProject.AI-Server) downloaded. That has all the code we've talked about above already in place.
2. Run the setup script in `/src`. This will ensure Python 3.9 is installed and setup, and that the required Python modules are installed.
3. Launch the server by starting a new debug session in Visual Studio or VS Code.
4. In Debug, the CodeProject.AI Server Dashboard is automatically launched when run. After the server starts, the dashboard will display the status of all the backend Modules, including the Background Removal module we just added. Notice in the image below, we've deliberately disabled some modules from starting for demonstration purposes.



    ![](https://raw.githubusercontent.com/ChrisMaunder/Adding-a-New-Module-to-CodeProject-AI-Server/master/docs/assets/rembg_started.JPG)
5. Launch the *index.html* file in a browser, choose a file and click **Submit** button. The results should be shown.



    ![Background Remover test.html page](https://raw.githubusercontent.com/ChrisMaunder/Adding-a-New-Module-to-CodeProject-AI-Server/master/docs/assets/test.html.jpg)

## What Next?

That's up to you. We've demonstrated a very simple AI module that removes the background from an image. The main work was

1. Ensuring you have the assets (eg models) available on a server so they can be downloaded
2. Updating the install script so your assets can be downloaded and moved into place, as well as ensuring you have the necessary runtime and libraries installed
3. Dropping in your module's code and writing an adapter so it can talk to the CodeProject.AI Server
4. Writing a *modulesettings* file that describes the API for your module
5. Testing! Always the fun part.

The possibilities on what you can add are almost limitless. Our goal is to enable you, as a developer, to add your own AI modules easily, and in turn get the benefit of modules that others have added. Mix and match, play with different sets of trained modules, experiment with settings and see where you can take this.

It's about learning and it's about having some fun. Go for it.

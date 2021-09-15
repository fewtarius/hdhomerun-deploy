# HDHomerun Deploy for ARM based NAS devices

## About

This is just a simple script that I wrote to install the HDHomerun Record software on my RN102 and DS418 NAS devices, and automatically deploy updates weekly.  It should work for other ARM based NAS devices too.

## Features

* Provisions HDHomerun Record
* Checks for and updates HDHomerun Record weekly

## License

Copyright 2018, Andrew Wyatt

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0
Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.

## Prerequisites

* Enable SSH on your NAS.
* A volume on your NAS for the HDHomerun software and recordings.

## Installation

* Log into the NAS with your user account.

* Download a zipped archive of this repository.

```
$ mkdir /yourvolume/hdhomerun
$ cd /yourvolume/hdhomerun
$ curl -Lo deploy_hdhomerun.tar.gz https://github.com/andrewwyatt/deploy_hdhomerun/archive/main.tar.gz
```

* Extract the archive and edit variables, changing BASEPATH to your storage volume.

```
$ tar -xvzf deploy_hdhomerun.tar.gz
$ cd deploy_hdhomerun-main
$ vi variables
```

* Run the deploy_homerun to perform the installation.

```
$ sudo ./deploy_hdhomerun
```

When completed, the HDHomerun Record software should be installed and running.

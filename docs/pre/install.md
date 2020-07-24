## Installation

Node can be run on Windows, MacOS, many "flavours" of Linux, Docker, etc. Almost any personal computer should have the necessary performance to run Node during development.

In this article we provide setup instructions for Windows, MacOS, and Ubuntu Linux.

## MacOS and Windows

Installing Node and NPM on Windows and MacOS is straightforward because you can just use the provided installer:

Download the required installer:
Go to [donwload page](https://nodejs.org/en/)
Select the button to download the LTS build that is "Recommended for most users".
Install Node by double-clicking on the downloaded file and following the installation prompts.

## Ubuntu 18.04

The easiest way to install the most recent LTS version of Node 10.x is to use the package manager to get it from the Ubuntu binary distributions repository. This can be done very simply by running the following two commands on your terminal:

```bash
curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
sudo apt-get install -y nodejs
```

## Testing your Nodejs and NPM installation
The easiest way to test that node is installed is to run the "version" command in your terminal/command prompt and check that a version string is returned:

```bash
> node -v
v12.13.0
```

The Nodejs package manager NPM should also have been installed, and can be tested in the same way:

```bash
> npm -v
6.9.0
```
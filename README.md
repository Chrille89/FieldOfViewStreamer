# FieldOfViewStreamer

The FieldOfViewStreamer contains on a server side (FieldOfViewStreamer) and a client side (FieldOfViewStreamerClient).
The server side is a nodejs web socket and transform the tiled video to a pure dash stream based on the HEVC-Tiles-Merger and push the segments to the client.
If the field of view (FoV) changes on the client side, the server must load the actual visible tile in higher resolution.

To deploy the server you need the following steps.

## Install NodeJS

```
wget --no-check-certificate https://nodejs.org/dist/v6.10.2/node-v6.10.2-linux-x64.tar.xz
tar -xvf node-v6.10.2.tar.gz
```

Now you can modify the PATH-Variable to use Node:

```
nano ~/.profile
```

and add the following lines of Code:

```
export PATH="$PATH:$HOME/node-v6.10.2-linux-x64/bin"
```

Now check if node is successfully installed by typing:
```
node -v
```
Now you have installed Nodejs and NPM on your Linux-Machine.

## Start the Server

Install dependencies:

```
cd FieldOfViewStreamer
npm install
node mediaStreamer.js
```

How to create the HEVC content you can see [here](https://gitlab.fokus.fraunhofer.de/fame-wm/hevc-js-tiles-merger).

To download and merge the content you must set the location of the hevc content in the config.json file.

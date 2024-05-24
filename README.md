[DigitalOcean Spaces](https://www.digitalocean.com/products/spaces) is an object storage service designed to provide a scalable and cost-effective solution for storing and serving files, such as images, videos, or any other static content. It is designed to make it easy for developers to store and manage large amounts of unstructured data. DigitalOcean Spaces is compatible with the Amazon S3 API for seamless integration with existing applications and tools. 

As a developer considering using Spaces for your unstructured data needs, you may be wondering just how performant it is and if it’s suitable for your use case. In this blog post, we’ll take a look at how much data you can ingest into a Spaces bucket through two different use cases to see what type of performance we can achieve.

## Prerequisites

If you would like to follow along with this tutorial, you’ll need the following:

* A DigitalOcean Account ([Sign Up for Free](https://try.digitalocean.com/freetrialoffer/) if you don’t already have one)
* A DigitalOcean Spaces Bucket ([Learn how to create one here](https://docs.digitalocean.com/products/spaces/how-to/create/))
* A recent version of Node.js installed on your development machine
* [Grafana k6 CLI](https://k6.io/open-source/) installed on your development machine

## Drone Delivery Tracker

To test the limits of Spaces, we wanted to build an app that you may find in the real world. For our sample application, we built a Drone Delivery Tracker. What the app does is simulate drones flying over Chicago and delivering packages. Each drone will send data back to homebase every second that includes the drone id, its location, a timestamp, the speed it’s traveling, as well as a Base64 encoded screenshot of what the drone is currently seeing. This way, the operator knows at all times how many drones are actively operating, where the drones are, what they’re seeing, and can keep track of the entire fleet.

All of the data that each drone generates takes up about **2.7mb** of storage and is stored in a text file. We will use DigitalOcean Spaces to permanently store this data. *The above data will of course be simulated and not real, but this is typically how IoT devices operate.* 

To get a better understanding of the scope of our application, let’s take a look at how we built it.

## Front-End

The front-end of our application displays a map of Chicago as well as any active drones in use. We will be using the [OpenLayers library](https://openlayers.org/) to display the map and our drones. We as the operators can spawn drones to simulate deliveries, and for each click of the “Add Drone” button, we’ll add one drone to our map that will start sending its whereabouts to our Spaces bucket every second. This means that if we have 1 drone, we’ll be sending 1 request each second to our back-end, which will take the data and push it to our DigitalOcean Spaces bucket. If we have 100 drones, we’ll be sending 100 requests each second for each drone, and so on and so forth.

![image](https://github.com/do-community/drone-tracker/assets/22687170/be77f992-92ed-46d2-9194-19d923a647ed)

*Due to limitations of how web browsers work, we are going to be throttled by the number of requests we can send each second. At the time of writing this post, Google Chrome limits the number of simultaneous requests to 10, while Mozilla Firefox limits the number of simultaneous requests to 17. Not to worry though, in our second example below, we’ll show how you can make many more requests manually with a load testing library.*

To understand how our code works on the front-end, let’s break it down. The majority of our work happens in the `main.js` file.

The first part of our app sets up and creates the OpenLayers map and points it to the Chicago metro area as well as creates a visual representation of a drone which will be represented by a blue dot. Additionally, we set the logic for how to render the drone on the map once rendering of the map is complete.

```js
import "./style.css";
import images from "./images";
import Map from "ol/Map.js";
import OSM from "ol/source/OSM.js";
import TileLayer from "ol/layer/Tile.js";
import View from "ol/View.js";
import { Circle as CircleStyle, Fill, Stroke, Icon, Style } from "ol/style.js";
import { Point } from "ol/geom.js";
import { getVectorContext } from "ol/render.js";
import { defaults as defaultInteractions } from "ol/interaction.js";

const tileLayer = new TileLayer({
  source: new OSM(),
});

const map = new Map({
  layers: [tileLayer],
  target: "map",
  view: new View({
    center: [-9790000, 5120000],
    zoom: 10,
  }),
  interactions: defaultInteractions({
    doubleClickZoom: false,
    dragAndDrop: false,
    dragPan: true,
    keyboardPan: false,
    keyboardZoom: false,
    mouseWheelZoom: true,
    pointer: true,
    select: true,
  }),
});

const imageStyle = new Style({
  image: new CircleStyle({
    radius: 5,
    fill: new Fill({ color: "blue" }),
    stroke: new Stroke({ color: "white", width: 1 }),
  }),
});


const drones = [];

tileLayer.on("postrender", function (event) {
  const vectorContext = getVectorContext(event);
  drones.forEach((drone) => {
    vectorContext.setStyle(imageStyle);
    vectorContext.drawGeometry(new Point([drone.x, drone.y]));
  });
});
```

Next, we implement the logic that will allow us to add drones to the map. We first add our buttons to add a single drone, as well as an ability to add 100 drones at once. We additionally capture the drone-count element in our HTML which will allow us to display the number of drones currently on the map. The `generateDrone`, `moveDrone`, and `sendDroneDataToSpaces` functions enable us to generate a unique drone when the add-drone button is clicked, move each unique drone to a random location, and finally send the movement information to our back-end so that it can be uploaded to DigitalOcean Spaces.

```js
let addDrone = document.getElementById("drones");
let addDrones = document.getElementById("add-drones");
let dronesDisplay = document.getElementById("drone-count");

addDrone.addEventListener("click", () => {
  setDrone();
});

addDrones.addEventListener("click", () => {
  for (let i = 0; i < 100; i++) {
    setDrone();
  }
});

function setDrone() {
  const drone = generateDrone();
  drones.push(drone);
}

let droneId = 1;
function generateDrone() {
  let drone = {
    id: droneId,
    x: -9795500,
    y: 5121000,
    ts: Date.now(),
    speed: Math.random() * 1000,
  };
  droneId += 1;
  return drone;
}

function moveDrones() {
  drones.forEach((drone) => {
    if (Math.random() > 0.5) {
      drone.x += 100 + drone.speed;
    } else {
      drone.x -= 100 - drone.speed;
    }
    if (Math.random() > 0.5) {
      drone.y += 100 + drone.speed;
    } else {
      drone.y -= 100 - drone.speed;
    }
    drone.ts = Date.now();
    drone.speed = Math.random() * 100;
    drone.image = images[0];
    sendDroneDataToSpaces(drone);
  });
}

async function sendDroneDataToSpaces(drone) {
  console.log("Sending Drone Data");
  console.log(drone);
  const formData = new FormData();
  formData.append("drone-id", drone.id);
  formData.append("image", drone.image);
  formData.append("speed", drone.speed);
  formData.append("x", drone.x);
  formData.append("y", drone.y);
  formData.append("timestamp", drone.ts);

  const res = await fetch("https://monkfish-app-m2nbx.ondigitalocean.app/", {
    method: "POST",

    body: formData,
  });
  const data = await res.json();
  console.log(data);
}
```
Finally, we set an interval to update all our information each second. Additionally, we allow the operator to stop sending drone data by clicking on the map.

```js
const interval = setInterval(() => {
  dronesDisplay.innerHTML = drones.length;
  moveDrones();
  map.render();
}, 1000);


map.on("click", function (evt) {
  clearInterval(interval);
});
```

Our HTML file to display the map and buttons is as follows:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link
      rel="icon"
      type="image/x-icon"
      href="https://openlayers.org/favicon.ico"
    />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Using OpenLayers with Vite</title>
    <script src="https://cdn.tailwindcss.com"></script>
  </head>
  <body>
    <div id="map"></div>
    <div class="absolute top-2 left-2 px-5 py-5 bg-white rounded-xl border-2">
      <h2 class="mb-2 font-bold">Menu</h2>
      <div class="bg-gray-200 p-2 rounded-lg mb-2">
        Drones: <span id="drone-count">0</span>
      </div>
      <button
        class="py-2 px-3 bg-green-500 rounded-lg text-sm text-white"
        id="drones"
      >
        Add A Drone
      </button>
      <button
        class="py-2 px-3 bg-green-500 rounded-lg text-sm text-white"
        id="add-drones"
      >
        Add Drones
      </button>
    </div>
    <script type="module" src="./main.js"></script>
  </body>
</html>
```

We use Vite to manage running our application, and to start the app up, simply run `npm install` to install all of the project dependencies, and then `npm start` to start up the application. By default, the app will run on `http://localhost:5173`, so navigate to that URL and you should see the app like so:

![image](https://github.com/do-community/drone-tracker/assets/22687170/f58298e3-38c6-4f73-8f59-6490dc9f96c4)

Next, let’s dive into the back-end of our application.

## Back-End

The back-end of our application will receive the drone location information and save it to our DigitalOcean Space. To create the Space from the project main page, click **Create** at the top of the page and select **Spaces** from the dropdown.

Select the desired datacenter to work from and choose a unique Spaces Bucket name. For the purposes of this tutorial, we'll choose `drone-tracker-manual` in the `sfo3` datacenter. Because this will act as internal storage and only be accessed with a separate API, we do not need to enable the CDN. Then click `Create Spaces Bucket`.

To create a back-end API, we used vanilla PHP deployed with [Digital Ocean App Platform](https://www.digitalocean.com/products/app-platform). App Platform will start a web server which will allow us to access our `.php` files as well as the `$_SERVER` and `$_POST` global variables.

### Application Logic

The full code for the backend can be found at [https://github.com/amycodes/drone-tracker](https://github.com/amycodes/drone-tracker) as well as instructions on how to install dependencies and configure associated application settings.

We will have a single post endpoint that will receive a json object. It will check for the following keys: `drone-id` and `timestamp`. The code as written will build a filename, then save the json objects to the `drone-tracker-manual` bucket using the S3Client in the AWS CLI SDK. If successful, it will then save the newly created `ObjectURL` from the response object and return that to the user.

#### Deploy to App Platform

To deploy our code to App Platform from the project main page, click **Create** at the top of the page and select **Apps** from the dropdown.

Select **GitHub** to populate the list of repositories, then select your fork of [https://github.com/amycodes/drone-tracker](https://github.com/amycodes/drone-tracker) from the dropdown. This will give us a list of branches to choose from as well as a **Source Directory**. For branch, choose **main**. For **Source Directory**, choose **backend/**. Click **Next**.

In the **Resources** summary, we can see our application as a Web Service and have the option to edit the service or the plan. Click **Edit Plan** below the list. Here, we can choose between Basic and Pro as well as our container size and number of containers. We will choose the Pro plan, and select 10 of the smallest containers available. This will ensure that we can handle more concurrent requests and as our app’s function is fairly simple, we don’t need a lot of resources per container. Click **Back** to return to the **Resources** page, then click **Next**. Here we can see our Environment Variables. We will need to add the following:

```
spaces_key: {SPACES_ACCESS_KEY}
spaces_secret: {SPACES_SECRET_KEY}
spaces_bucket: drone-tracker-manual
spaces_endpoint: https://sfo.digitaloceanspaces.com
```

Click **Encrypt** for `spaces-secret`. Click Save. To continue, click **Next**.

The next screen will give you a review of your App Platform Info. Here we can change the name of our App, our associated Project, and Region. Click **Next**.

The final screen will provide an overview of Resources, App Info, and Billing Details. Click **Create Resources**.

This will deploy our web service resources. To ensure this service is accessible from the front-end we will navigate to the **Settings** tab. We will select our web service from the App Platform Components and scroll down to **CORS Policy** and click **Edit**. Next, click **Edit CORS Policy**. This will give you a place to add access control policies. Under **Access-Control-Allow-Origins**, click **Regex**.Under origin, input `.*`. Click **Next: Review** then **Save**.

Now we can post to our web service and have it saved to our DigitalOcean Space.

## Manually Testing DigitalOcean Spaces Throughput

Our Drone Delivery Tracker app is a fun visual representation that showcases how you can use DigitalOcean Spaces to store unstructured data, but due to browser limitations doesn’t fully allow us to stress test how quickly we can add data to a DigitalOcean Spaces bucket.

To address that, we also built a Node.js script that further simulates and allows you to send the same data without browser-based limitations. If you look in the ‘drone-tracker-manual` directory in the GitHub repository, you will see that code. Let’s take a closer look at it.

This directory has 3 files. A `drone.txt` file that represents the data a Drone in our previous application generates each second. A `load.js` file, which we’ll use for our load testing. And an `index.js` file that has our implementation. Let’s open up our `index.js` file. 

```js
const { S3Client, PutObjectCommand } = require("@aws-sdk/client-s3");
const { createId } = require("@paralleldrive/cuid2");
const fs = require("fs");
const express = require("express");
const app = express();
```

To break this code down, we are first importing our dependencies. Which are the AWS S3 SDK, which will allow us to upload files to Spaces. Since DigitalOcean Spaces is an S3 compatible storage solution, we can leverage the AWS S3 SDK directly. Next cuid2 will allow us to generate random ids for our drones, express will allow us to easily setup a Node.js web server, and finally k6 will allow us to stress test the application and send it many hundreds of requests per second.

```js
// Replaces with your Spaces access key and secret key
const ACCESS_KEY = "YOUR-SPACES-ACCESS-KEY";
const SECRET_KEY = "YOUR-SPACES-ACCESS-SECRET";

// Creates a client for the Spaces service
const client = new S3Client({
  region: "sfo3",
  endpoint: "https://sfo3.digitaloceanspaces.com",
  credentials: {
    accessKeyId: ACCESS_KEY,
    secretAccessKey: SECRET_KEY,
  },
});

// Set the name of your bucket and the file you want to upload
const BUCKET_NAME = "drone-tracker-manual";
const FILE_NAME = "./drone.txt";

// Reads the file into a Buffer
const fileBuffer = fs.readFileSync(FILE_NAME);
```

With our dependencies out of the way, let’s setup the connection to our DigitalOcean Spaces bucket. We’ll use the AWS S3Client to do this. Note that we’ll set the region to our DigitalOcean Spaces region where we created our bucket, as well as the endpoint will point to `digitaloceanspaces.com`.

Next, we’ll define which bucket we want to store our data in, and we’ll read the `drone.txt` which will have the data that we want to upload.

```js
app.get("/", async (req, res) => {
  let index = createId();

  const params = {
    Bucket: BUCKET_NAME,
    Key: `${index}.txt`,
    Body: fileBuffer,
  };
  const result = await client.send(new PutObjectCommand(params));
  res.json(result);
});

app.listen("3000", () => {
  console.log(`Example app listening on port: 3000`);
});
```

Finally, we’ll set up our Express server. We’ll have just one route that we’ll be able to access via `http://localhost:3000/`. When this route is called, we will generate a random id, get the `drone.txt` file, and upload it to our DigitalOcean Spaces bucket called `drone-tracker-manual`. Every time we hit this endpoint, we’ll upload another file. To start up this application, in your terminal run `node index.js`.

## Load Testing with Grafana k6

While we can manually hit this endpoint from a browser over and over again, that wouldn’t be very efficient for stress testing. That’s why we’ll use Grafana k6, an open source load testing library.

If you open up the `load.js` file in the Github repository, you will find this code:

```js
import http from "k6/http";

export default function () {
  http.get("http://localhost:3000");
}
```

What this code does is it simply calls our `http://localhost:3000` endpoint, which uploads a single file to DigitalOcean Spaces. To run this code, in your terminal navigate to the Github repository where the `load.js` file is and run `k6 load.js`. If the script run successfully, you’ll see an output that looks like this:

![image](https://github.com/do-community/drone-tracker/assets/22687170/6c81dde7-c9ba-4e14-b509-80edb50d3cac)

One request is good, but we want to load test our app. To increase the number of requests we can pass in the `--vus` parameter which will simulate additional users. Let’s run `k6 run --vus 100 --duration 15s load.js` to simulate 100 users hitting our endpoint consistently over 15 seconds. The output of this command shows that we hit the `http://localhost:3000` endpoint 663 times in 15 seconds, meaning that we uploaded 663 files to our DigitalOcean Spaces bucket in that time. 

![image](https://github.com/do-community/drone-tracker/assets/22687170/ab815428-44d2-4587-84fb-262a6759a313)

Let’s try it again, but this time we’ll do 500 users over a 60 second time period to show both concurrent requests and sustained load. In my test, I was able to upload 2899 files amounting to over 8GB of data to Spaces over a 60 second time period. *Do note that my limitation here is my computer running Node.js.* If our application was running in the cloud in a more distributed manner, we could greatly increase the throughput.

DigitalOcean Spaces currently supports up to 800 requests per second to handle almost any use case and is a powerful and flexible solution for developers looking to store and serve large amounts of static content. By following best practices and understanding its key features, you can effectively scale your storage needs and optimize performance as your applications grow.

Happy building!

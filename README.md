### PAD aditional task

This code is an **API Gateway** for a microservices-based system, built using **Express.js**. The gateway routes requests to different backend services while handling **load balancing, caching with Redis, and logging with Winston and Logstash**.

---

## **1. Dependencies and Setup**
```js
const express = require("express");
const axios = require("axios");
const redis = require("redis");
const winston = require('winston');
```
- **Express.js** – Web framework for handling HTTP requests.
- **Axios** – For making HTTP requests to backend services.
- **Redis** – Used for caching API responses.
- **Winston** – Logging library for logging to Logstash.

```js
require('dotenv').config();
const LOGGING = parseInt(process.env.LOGGING);
const LOGSTASH_HOST = process.env.LOGSTASH_HOST;
const LOGSTASH_PORT = process.env.LOGSTASH_PORT;
```
- Loads environment variables for **logging configuration**.

---

## **2. Logging Setup**
```js
const infoLogger = winston.createLogger({
    transports: [
        new winston.transports.Http({
            host: LOGSTASH_HOST,
            port: LOGSTASH_PORT,
            level: 'info',
        })
    ],
});

const errLogger = winston.createLogger({
    transports: [
        new winston.transports.Http({
            host: LOGSTASH_HOST,
            port: LOGSTASH_PORT,
            level: 'error'
        })
    ]
});
```
- Sets up **two loggers**: one for **info** logs, another for **error** logs.
- Logs are sent to **Logstash** for centralized logging.

```js
function logMsg(msg) {
    if (LOGGING) {
        infoLogger.info(JSON.stringify({
            "service": "gateway",
            "msg": msg
        }));
    }
}

function logErr(msg) {
    if (LOGGING) {
        errLogger.error(JSON.stringify({
            "service": "gateway",
            "error": msg
        }));
    }
}
```
- Helper functions to log messages **only if logging is enabled**.

---

## **3. Redis Caching Setup**
```js
const redisClient = redis.createClient({ url: process.env.REDIS_URL || "redis://gateway-cache:6379" });
redisClient.connect();
```
- Connects to a **Redis cache** (defaulting to `gateway-cache:6379`).
- Used to cache API responses to improve performance.

---

## **4. Load Balancing Setup**
```js
const chatBases = ["http://chat-1:8000", "http://chat-2:8000", "http://chat-3:8000"];
const assetBases = ["http://asset-1:8000", "http://asset-2:8000", "http://asset-3:8000"];
let chatIndex = 0;
let assetIndex = 0;
```
- Defines backend **Chat Service** and **Asset Service** instances.
- Maintains **round-robin load balancing indexes**.

```js
const getNextHost = (serviceHosts, index) => {
    const host = serviceHosts[index % serviceHosts.length];
    return { host, newIndex: (index + 1) % serviceHosts.length };
};
```
- **Round-robin algorithm** to distribute requests across multiple instances.

---

## **5. Caching Mechanism**
```js
const cachingBlacklist = [
    "/chatApp/utilities/status/",
    "/assetApp/utilities/status/",
    "/chats/"
];
```
- **Excludes certain API routes** from caching.

```js
const getCachingKey = (req) => {
    if (
        req.method !== "GET" || 
        cachingBlacklist.some((path) => req.originalUrl.startsWith(path))
    ) {
        return null;
    }

    let cachingKey = `get:${req.originalUrl}`;
    const username = req.headers["x-username"];

    if (username) {
        cachingKey += `:username=${username}`;
    }

    return cachingKey;
};
```
- **Only caches GET requests**.
- **Generates a caching key** based on the request URL and username (if present).

---

## **6. Handling Requests**
```js
const handleRequest = async (req, res, serviceHosts, indexTracker) => {
    const { host, newIndex } = getNextHost(serviceHosts, indexTracker);
    const cachingKey = getCachingKey(req);
```
- Selects a backend **service instance** using round-robin.
- Generates a **caching key** if caching is applicable.

```js
    if (cachingKey) {
        try {
            const cachedResponse = await redisClient.get(cachingKey);
            if (cachedResponse) {
                logMsg(`LOG: Retrieving ${req.method}:${req.originalUrl} from cache`);
                return res.json(JSON.parse(cachedResponse));
            }
        } catch (error) {
            console.error("Redis error:", error);
        }
    }
```
- **Checks Redis cache** before making a backend request.

```js
    logMsg(`LOG: Requesting ${req.method}:${req.originalUrl} from ${host}`);

    const url = `${host}${req.originalUrl}`;
    try {
        const response = await axios({
            method: req.method,
            url: url,
            data: req.body,
            headers: req.headers
        });

        if (cachingKey) {
            await redisClient.setEx(cachingKey, 60, JSON.stringify(response.data));
        }

        res.status(response.status).json(response.data);
    } catch (error) {
        logErr(`ERR: Received ${error.response?.status || 500} response from ${new URL(url).hostname.split(':')[0]}`);

        res.status(error.response?.status || 500).json(error.response?.data || { detail: "Internal Server Error" });
    }
```
- **Makes a request** to the selected backend service.
- **Stores successful responses in Redis** (cache expiration: 60s).
- Logs and returns **errors appropriately**.

```js
    if (serviceHosts === chatBases) {
        chatIndex = newIndex;
    } else {
        assetIndex = newIndex;
    }
};
```
- Updates the **round-robin index** after each request.

---

## **7. Defining API Routes**
```js
const chatEndpoits = [
    "/chats/",
    "/chats/:chat_id/",
    "/chats/unseen-list/",
    "/messages/send-to-chat/:chat_id/",
    "/messages/:message_id/",
    "/messages/:message_id/mark-as-seen/",
    "/chatApp/utilities/status/"
];
const assetEndpoints = [
    "/assets/",
    "/assets/:asset_id/",
    "/assets/:asset_id/add-to-scene/:scene_id/",
    "/assets/list-for-scene/:scene_id/",
    "/assets/get-file-url/:asset_id/",
    "/scenes/",
    "/scenes/:scene_id/",
    "/assetApp/utilities/status/",
];

chatEndpoits.forEach((route) => {
    app.all(route, (req, res) => handleRequest(req, res, chatBases, chatIndex));
});

assetEndpoints.forEach((route) => {
    app.all(route, (req, res) => handleRequest(req, res, assetBases, assetIndex));
});
```
- **Maps API routes** to their corresponding backend services.
- Uses `app.all()` to support **GET, POST, PUT, DELETE** requests.

---

## **8. API Gateway Status Route**
```js
app.get('/status/', (req, res) => {
    res.status(200).json({ message: `API Gateway at 127.0.0.1:${PORT} is up and running` });
});
```
- **Health check** route for the API Gateway.

---

## **9. Start the Server**
```js
const PORT = 8080;
app.listen(PORT, () => {
    console.log(`Gateway running at 127.0.0.1:${PORT}`);
});
```
---
## assets/view.py
This code defines several API views for handling **assets** and their relationship with **scenes** using Django Rest Framework (DRF). Let’s break it down step by step.


### **Imports**
```python
from rest_framework.generics import GenericAPIView
from rest_framework.response import Response
from rest_framework import status
from .models import Asset
from scenes.models import Scene
from .serializers import AssetListSerializer, AssetSerializer
from rest_framework.views import APIView
```
- **GenericAPIView**: A DRF class-based view that provides helper methods for common functionality (like `get_queryset`, `get_serializer`).
- **Response**: Used to send JSON responses.
- **status**: Provides HTTP status codes (`200 OK`, `404 Not Found`, etc.).
- **Asset & Scene models**: ORM models representing assets and scenes in the database.
- **Serializers**:
  - `AssetSerializer`: Serializes an individual asset.
  - `AssetListSerializer`: Serializes a list of assets.
- **APIView**: A more basic class-based view that doesn’t assume any particular model.

---

## **1. AssetView (CRUD operations for Assets)**
```python
class AssetView(GenericAPIView):
    queryset = Asset.objects.all()
    serializer_class = AssetSerializer
```
- `queryset = Asset.objects.all()`: Defines the queryset of all `Asset` objects.
- `serializer_class = AssetSerializer`: Uses `AssetSerializer` for serialization.

### **GET: Retrieve Assets (Single or List)**
```python
def get(self, request, pk=None):
    if pk:
        instance = self.get_object()
        serializer = self.get_serializer(instance)
    else:
        instances = self.get_queryset()
        serializer = AssetListSerializer(instances, many=True)

    return Response(serializer.data, status=status.HTTP_200_OK)
```
- If a primary key (`pk`) is provided → retrieve a **single asset**.
- If no `pk` is provided → retrieve **all assets**.
- Uses `AssetListSerializer` when returning multiple assets.

### **POST: Create a New Asset**
```python
def post(self, request):
    serializer = self.get_serializer(data=request.data)
    if serializer.is_valid(raise_exception=True):
        serializer.save()
        return Response(serializer.data, status=status.HTTP_201_CREATED)
    
    return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```
- Validates request data and creates an **Asset** if valid.
- Returns `201 Created` if successful.

### **PATCH: Update an Existing Asset**
```python
def patch(self, request, pk):
    instance = self.get_object()
    serializer = self.get_serializer(instance, data=request.data, partial=True)
    if serializer.is_valid(raise_exception=True):
        serializer.save()
        return Response(serializer.data, status=status.HTTP_200_OK)

    return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```
- Retrieves the asset by `pk` and updates it **partially**.
- Uses `partial=True`, meaning only provided fields will be updated.

### **DELETE: Remove an Asset**
```python
def delete(self, request, pk):
    instance = self.get_object()
    instance.delete()
    
    return Response(status=status.HTTP_204_NO_CONTENT)
```
- Retrieves an asset by `pk` and deletes it.
- Returns `204 No Content` (successful deletion, no response body).

---

## **2. AddAssetToSceneView (Add an Asset to a Scene)**
```python
class AddAssetToSceneView(APIView):
    def post(self, request, pk, scene_id):
        try:
            scene_instance = Scene.objects.get(id=scene_id)
            asset_instance = Asset.objects.get(id=pk)
        except Scene.DoesNotExist:
            return Response({'detail': 'Scene not found.'}, status=status.HTTP_404_NOT_FOUND)
        except Asset.DoesNotExist:
            return Response({'detail': 'Asset not found.'}, status=status.HTTP_404_NOT_FOUND)
        
        scene_instance.assets.add(asset_instance)

        return Response(
            {'message': f'Asset#{pk} added to scene#{scene_id} successfully.'},
            status=status.HTTP_200_OK
        )
```
- Retrieves the **Scene** and **Asset** by their IDs.
- If either doesn’t exist → returns `404 Not Found`.
- If both exist → adds the asset to the scene (`scene_instance.assets.add(asset_instance)`).
- Returns `200 OK` with a success message.

---

## **3. ListAssetsForScene (Get All Assets in a Scene)**
```python
class ListAssetsForScene(GenericAPIView):
    queryset = Asset.objects.all()
    serializer_class = AssetListSerializer

    def get(self, request, scene_id):
        try:
            scene_instance = Scene.objects.get(id=scene_id)
        except Scene.DoesNotExist:
            return Response({'detail': 'Scene not found.'}, status=status.HTTP_404_NOT_FOUND)
        
        instaces = Asset.objects.filter(scene=scene_instance)
        serializer = self.get_serializer(instaces, many=True)

        return Response(serializer.data, status=status.HTTP_200_OK)
```
- Fetches all assets related to a specific scene.
- Returns `404` if the scene doesn’t exist.
- Uses `AssetListSerializer` to return asset details.

---

## **4. GetFileUrlView (Get File URL for an Asset)**
```python
class GetFileUrlView(APIView):
    def get(self, request, pk):
        try:
            instance = Asset.objects.get(id=pk)
        except Asset.DoesNotExist:
            return Response({'detail': 'Asset not found.'}, status=status.HTTP_404_NOT_FOUND)
        
        return Response({'file_url': instance.file_url}, status=status.HTTP_200_OK)
```
- Retrieves an **Asset** by `pk`.
- If found → returns its `file_url`.
- If not found → returns `404 Not Found`.

---

## **Summary**
### **AssetView**
- Handles **CRUD** operations on `Asset` objects.
  - `GET /assets/` → List all assets.
  - `GET /assets/{pk}/` → Retrieve a single asset.
  - `POST /assets/` → Create a new asset.
  - `PATCH /assets/{pk}/` → Partially update an asset.
  - `DELETE /assets/{pk}/` → Delete an asset.

### **AddAssetToSceneView**
- `POST /assets/{pk}/add_to_scene/{scene_id}/` → Adds an asset to a scene.

### **ListAssetsForScene**
- `GET /scenes/{scene_id}/assets/` → Lists all assets in a scene.

### **GetFileUrlView**
- `GET /assets/{pk}/file_url/` → Gets the file URL of an asset.

This setup provides a **well-structured API** for managing **assets** and their **scene associations**! 🚀
- Starts the **Express server** on port **8080**.

---


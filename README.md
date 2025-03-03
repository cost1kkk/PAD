# index.js (gatway)

This code is an API Gateway for a microservices-based system, built using Express.js. The gateway routes requests to different backend services while handling load balancing, caching with Redis, and logging with Winston and Logstash.
#1 Dependencies and Setup

```JavaScript
const express = require("express");
const axios = require("axios");
const redis = require("redis");
const winston = require('winston');

```
Express.js – Web framework for handling HTTP requests.
Axios – For making HTTP requests to backend services.
Redis – Used for caching API responses.
Winston – Logging library for logging to Logstash.

```JavaScript
require('dotenv').config();
const LOGGING = parseInt(process.env.LOGGING);
const LOGSTASH_HOST = process.env.LOGSTASH_HOST;
const LOGSTASH_PORT = process.env.LOGSTASH_PORT;
```
Loads environment variables for logging configuration.

#2 Logging Setup


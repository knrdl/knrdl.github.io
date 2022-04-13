---
title: "Arduino: ESP as static webserver"
date: "2022-04-13"
tags: ["arduino", "c++", "webserver"]
lang: "en"
---

The ESP8266/ESP32/NodeMCU boards firmware has builtin support to run a http server. It's possible to build a restful
json api but also to serve static files.

# Setup

Arduino IDE → File → Preferences → Additional Board Manager
URLs: http://arduino.esp8266.com/stable/package_esp8266com_index.json

# Connect to Wifi

```c++
#include <ESP8266WiFi.h>
#include <WiFiClient.h>

const char *ssid = "YOUR_NETWORK_NAME";
const char *password = "YOUR_NETWORK_PASSWORD";

void setup(void) {
  WiFi.mode(WIFI_STA);  // stationary mode
  WiFi.begin(ssid, password);

  // Wait for connection
  while (WiFi.status() != WL_CONNECTED) {
    delay(50);
  }
}

void loop(void) {
}
```

# Basic API Server

```c++
#include <ESP8266WiFi.h>
#include <WiFiClient.h>
#include <ESP8266WebServer.h>

ESP8266WebServer server(80);

inline void handleNotFound() {
  server.send(404, "text/plain", "404 not found");
}

inline void serveInfo() {
  String json = "{";
  json += "\"free_heap\":" + String(ESP.getFreeHeap());
  json += ",\"heap_frag_perc\":" + String(ESP.getHeapFragmentation());
  json += ",\"last_reset_reason\":\"" + String(ESP.getResetReason()) + "\"";
  json += ",\"uptime\":" + String(millis() / 1000);
  json += ",\"cpu_freq\":" + String(ESP.getCpuFreqMHz());
  json += ",\"wifi_rssi\":" + String(WiFi.RSSI());
  json += ",\"ip\":\"" + WiFi.localIP().toString() + "\"";
  json += ",\"mac\":\"" + WiFi.macAddress() + "\"";
  json += "}";
  server.send(200, "application/json", json);
}

void setup(void) {
  server.on("/api/system", serveInfo);
  server.onNotFound(handleNotFound);
  server.begin();
}

void loop(void) {
  server.handleClient();
}
```

# Static file server

There a few more steps to take here:

1. Aggregate files which should be served by the arduino
2. Flash files from computer to a filesystem on the arduino board
3. Serve files from the arduino filesystem via http.

## LittleFS

LittleFS is a filesystem for microcontrollers. It supports path-based read/write operations:

```c++
#include "LittleFS.h"

inline void writeFile(const String path, const String message) {
  File file = LittleFS.open(path, "w");
  file.print(message);
  file.close();
}

inline String readFile(const String path) {
  File file = LittleFS.open(path, "r");
  if (!file || file.isDirectory()) {
    return "";
  }
  const String message = file.readString();
  file.close();
  return message;
}
```

## Fileserver

A webserver to serve files from a single directory (e.g. `/index.html`, `/styles.css`, `/main.js`, ...) based on
LittleFS would look like this:

```c++ {linenos=table, hl_lines=["17-19"]}
#include <ESP8266WebServer.h>
#include "LittleFS.h"
#include <uri/UriBraces.h>

ESP8266WebServer server(80);

inline void serveStaticFiles() {
  String filename = server.pathArg(0);
  filename.replace("..", "");  // mitigate path traversal attacks
  filename.replace("/", "");  // serve from a single folder only
  filename.trim();
  if (filename == "") {
    filename = "index.html";
  }
  String path = "/www/" + filename;  // assuming files are served from /www/
  const String contentType = mime::getContentType(path);
  if (!LittleFS.exists(path)) {
    path = path + ".gz";  // explained in compression section below
  }
  if (!LittleFS.exists(path)) {
    server.send(404, "text/plain", "404 not found");
  } else {
    File file = LittleFS.open(path, "r");
    server.streamFile(file, contentType);
    file.close();
  }
}

void setup(void) {
  LittleFS.begin();

  server.on(UriBraces("/{}"), serveStaticFiles);
  server.begin();
}

void loop(void) {
  server.handleClient();
}
```

## Setup

To flash files to the arduino an additional library for Arduino IDE is required:

1. https://github.com/earlephilhower/arduino-esp8266littlefs-plugin
2. unpack to `~/Arduino/tools/ESP8266LittleFS/tool/esp8266littlefs.jar`
3. Restart Arduino IDE
4. In Arduino IDE: Tools → ESP8266 Little FS Data Upload
5. It will upload all files (and dirs) in the `data` directory, which needs to be in the same folder as the
   sketch (`.ino`
   file)

## Compression

The ESP8266 runs at 80MHz cpu frequency (can be changed to 160MHz). Transferring large files takes some time (e.g. more
than a second for a couple of megabytes).

So pre-compressing files might improve performance. The following script gzips all files in the `www` dir and saves them
to `data/www`:

```shell
#!/bin/bash
rm -rf data/www
mkdir -p data/www
for filename in ./www/*; do
   cat "$filename" | gzip -9 > "data/$filename.gz"
done
```

So when using the "ESP8266 Little FS Data Upload" button only the compressed versions will be uploaded (as `/www/...` on
the arduino). The server can handle pre-compressed files because of the precaution in Lines 17-19 of the server script
above. The correct http response header `Content-Encoding: gzip` will be automatically set by the command on Line 24.

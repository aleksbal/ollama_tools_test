# Experimenting with Ollama's REST API for Tool Integration

## Introduction

Ollama 3.2 provides a powerful REST API for interacting with its language models. While it also supports Python and JavaScript APIs, I explored direct REST communication using **Insomnia**, a popular API client. My goal was to test how well Ollama could leverage external tools to answer real-world queries requiring current and location-based information.

## Experiment Setup

To begin, I initiated a chat session with Ollama via its REST API, defining two tools:

1. **get_current_weather** – A function to fetch real-time weather data for a given location.
2. **get_geojson** – A function to retrieve geospatial data for a location.

Given that large language models (LLMs) lack real-time access to external data, I wanted to assess whether Ollama would correctly infer that both tools should be used to answer my query.

## Initial Request

I prompted Ollama with the following question:

> *What is the weather today in the capital of France and where is the city on the map? Be concise, don't make things up!*

The API request included the model name, chat history, and the tool definitions:

```json
{
  "model": "llama3.2",
  "messages": [
    {
      "role": "user",
      "content": "What is the weather today in the capital of France and where is the city on the map? Be concise, don't make things out!"
    }
  ],
  "stream": false,
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_current_weather",
        "description": "Get the current weather for a location",
        "parameters": {
          "type": "object",
          "properties": {
            "location": {
              "type": "string",
              "description": "The location to get the weather for, e.g. San Francisco, CA"
            }
          },
          "required": ["location"]
        }
      }
    },
    {
      "type": "function",
      "function": {
        "name": "get_geojson",
        "description": "Geospatial data for locations like city, river, region, mountain, state, or other landmark",
        "parameters": {
          "type": "object",
          "properties": {
            "name": {
              "type": "string",
              "description": "The location name to get the GeoJSON for"
            }
          },
          "format": {
              "type": "FutureCollection",
              "description": "Collection of GeoJSON Features (Points, Polygons, LineStrings)"
          },
          "required": ["name"]
        }
      }
    }    
  ]
}
```

## Initial Response from Ollama

Since Ollama itself does not have real-time access to weather or geospatial data, it did not fabricate an answer. Instead, it correctly inferred that it needed to invoke the external tools and generated a response indicating which functions should be called:

```json
{
  "model": "llama3.2",
  "message": {
    "role": "assistant",
    "content": "",
    "tool_calls": [
      {
        "function": {
          "name": "get_current_weather",
          "arguments": {
            "location": "Paris, France"
          }
        }
      },
      {
        "function": {
          "name": "get_geojson",
          "arguments": {
            "name": "Paris"
          }
        }
      }
    ]
  },
  "done_reason": "stop"
}
```

This was a great validation that Ollama correctly understood the request and mapped it to the appropriate tools without explicit prompting.

## Second Request to Ollama with Tool Results

After obtaining the results from the tool invocations, I sent another request to Ollama including these results:

```json
{
  "model": "llama3.2",
  "stream": false,
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_current_weather",
        "description": "Get the current weather for a location",
        "parameters": {
          "type": "object",
          "properties": {
            "location": {
              "type": "string",
              "description": "The location to get the weather for, e.g. San Francisco, CA"
            }
          },
          "required": ["location"]
        }
      }
    },
    {
      "type": "function",
      "function": {
        "name": "get_geojson",
        "description": "Geospatial data for location like city, river, region, mountain, state or other landmark",
        "parameters": {
          "type": "object",
          "properties": {
            "name": {
              "type": "string",
              "description": "The location name to get the GeoJSON for"
            }
          },
          "format": {
              "type": "FutureCollection",
              "description": "Collection of GeoJSON Features (Points, Polygons, LineStrings)"
          },
          "required": ["name"]
        }
      }
    }
  ],
  "messages": [
    {
      "role": "user",
      "content": "What is the weather today in the capital of France and where is the city on the map? Be concise, don't make things out!"
    },
    {
      "role": "tool",
      "content": "Very cold, snow, some wind"
    },
    {
      "role": "tool",
      "content": "{ \"type\": \"FeatureCollection\", \"features\": [ { \"type\": \"Feature\", \"geometry\": { \"type\": \"Point\", \"coordinates\": [2.39007, 48.8257] }, \"properties\": { \"code\": \"75001\", \"nom\": \"Paris\" } } ] }"
    }
  ]
}
```

## Final Response

After receiving the updated request with tool results, Ollama generated the following response:

```json
{
  "model": "llama3.2",
  "message": {
    "role": "assistant",
    "content": "The current weather in Paris, the capital of France is very cold with snow and some wind. Paris is located at a point on the map with coordinates [2.39007, 48.8257]."
  },
  "done_reason": "stop"
}
```


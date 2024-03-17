---
layout: post
title: "Singular Update Queue"
categories: misc
---

### Using a single thread to process requests asynchronously to maintain order without blocking the caller

### Problem

When the state needs to be updated by multiple concurrent clients, we need it to be safely updated with one-at-a-time changes. We need entries to be processed one at a time, even if several concurrent clients are trying to write. Generally, locks are used to protect against concurrent modifications. But if the tasks being performed are time-consuming, such as writing to a file, blocking all the other calling threads until the task is completed can have severe impact on the overall system throughput and latency. It's important to make effective use of compute resources, while still maintaining the guarantee of one-at-a-time execution.

### Solution

Implement a work queue and a single thread working off the queue. Multiple concurrent clients can submit state changes to the queue - but a single thread works on state changes. This can be naturally implemented with goroutines and channels in languages like Go.

## Implementation

This can be a natural fit for languages or libraries supporting lightweight threads along with the concept of channels(such as Go or Kotlin), All the requests are passed to a single channel to be processed. In Go, there is a single goroutine which processes all the messages to update state. The response is then written to a separate channel and processed by a separate goroutine to send it back to clients as seen in the following code, the requests to update key value are passed onto a single shared request channel:

The request are processed in a single goroutine to update all the states.

```go
package main

import (
	"encoding/json"
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
	"time"

	"github.com/go-chi/chi"
)

var (
	requestChannel  chan string
	responseChannel chan string
)

func init() {
	requestChannel = make(chan string)
	responseChannel = make(chan string)
}

func main() {

	r := chi.NewRouter()
	r.Post("/", putKV)

	go singularUpdateQueue()

	http.ListenAndServe(":8080", r)
}

func putKV(w http.ResponseWriter, r *http.Request) {

	reqBody, _ := ioutil.ReadAll(r.Body)

	requestChannel <- string(reqBody)

	select {
	case msg := <-responseChannel:
		fmt.Println("Got value from channel:", msg)
		w.Write([]byte(msg))
	case <-time.After(1 * time.Second):
		fmt.Println("Timed out!")
	}

}

func singularUpdateQueue() {
	for {
		select {
		case e := <-requestChannel:
			mapResponse := map[string]string{"Status": "OK", "RequestBody": e}
			response, err := json.Marshal(mapResponse)
			if err != nil {
				log.Fatal(err)
			}
			responseChannel <- string(response)
		}
	}
}
```

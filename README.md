# Embeddable UI:

# Concept

- iframes communicate with the parent window via postMessage
- the parent window can send messages to the iframe
- the iframe can send messages to the parent window

# Communication Protocol

## Message Structure

```typescript
type Message = {
  type: string;
  messageId?: string; // optional, used for tracking the message
  payload: Record<string, unknown>;
};
```

## Message Types

### `intent`

- indicates that the user has interacted with the UI and expressed an intent, and the host should act on it
- the payload is an object with the following properties:
  - `intent` - the intent that the user expressed
  - `params` - the parameters to pass to the intent

**Example:**

```typescript
window.parent.postMessage(
  {
    type: "intent",
    payload: {
      intent: "create-task",
      params: {
        title: "Buy groceries",
        description: "Buy groceries for the week",
      },
    },
  },
  "*"
);
```

### `notify`

- indicates that the iframe already acted upon the user interaction, and is notifying the host to trigger any side effects
- the payload is an object with the following properties:
  - `message` - the message to notify the host with

**Example:**

```typescript
window.parent.postMessage(
  {
    type: "notify",
    payload: {
      message: "cart-updated",
    },
  },
  "*"
);
```

### `prompt`

- indicates that the iframe asks the host to run a prompt
- the payload is an object with the following properties:
  - `prompt` - the prompt to run

**Example:**

```typescript
window.parent.postMessage(
  {
    type: "prompt",
    payload: {
      prompt: "What is the weather in Tokyo?",
    },
  },
  "*"
);
```

### `tool`

- indicates that the iframe asks the host to run a tool call
- the payload is an object with the following properties:
  - `toolName` - the name of the tool to run
  - `params` - the parameters to pass to the tool

**Example:**

```typescript
window.parent.postMessage(
  {
    type: "tool",
    payload: {
      toolName: "get-weather",
      params: {
        city: "Tokyo",
      },
    },
  },
  "*"
);
```

### `link`

- indicates that the iframe asks the host to navigate to a link
- the payload is an object with the following properties:
  - `url` - the URL to navigate to

**Example:**

```typescript
window.parent.postMessage(
  {
    type: "link",
    payload: {
      url: "https://www.google.com",
    },
  },
  "*"
);
```

## Reserved Message Types - from the iframe to the host

### `ui-lifecycle-iframe-ready`

- indicates that the iframe is ready to receive messages

**Example:**
See [Render Data](#render-data)

### `ui-size-change`

- indicates that the iframe's size has changed and the host should adjust the iframe's size
- the payload is an object with the following properties:
  - `width` - the new width of the iframe
  - `height` - the new height of the iframe

**Example:**

```typescript
const resizeObserver = new ResizeObserver((entries) => {
  entries.forEach((entry) => {
    window.parent.postMessage(
      {
        type: "ui-size-change",
        payload: {
          height: entry.contentRect.height,
        },
      },
      "*"
    );
  });
});

resizeObserver.observe(document.documentElement);
```

### `ui-request-data`

- a message that the iframe sends to the host to request data. The message must include a `messageId` to allow the iframe to track the response.
- the payload is an object with the following properties:
  - `requestType` - the type of the request
  - `params` - the parameters to pass to the request

**Example:**

```typescript
window.parent.postMessage(
  {
    type: "ui-request-data",
    messageId: "123",
    payload: {
      requestType: "get-payment-methods",
      params: {
        // any params needed for the request
      },
    },
  },
  "*"
);
```

See also [Asynchronous Data Requests with Message IDs](#asynchronous-data-requests-with-message-ids)

## Reserved Message Types - from the host to the iframe

### `ui-lifecycle-iframe-render-data`

- a message that the host sends to the iframe to pass any relevant render data
- the payload is an object with the following properties:
  - `renderData` - the render data to pass to the iframe

**Example:**
See [Render Data](#render-data)

### `ui-action-received`

- a message that the host sends to the iframe to indicate that the action has been received. The original messageId is passed back to the host to allow the host to track the action. This is useful for `request-data` messages, but is not limited to this type.
  **Example:**
  See [Asynchronous Data Requests with Message IDs](#asynchronous-data-requests-with-message-ids)

### `ui-action-response`

- a message that the iframe sends to the host to indicate that the action has been processed. The original messageId is passed back to the host to allow the host to track the action. This is useful for `request-data` messages, but is not limited to this type.
- the payload is an object with the following properties:
  - `response` - the response to the action

**Example:**
See [Asynchronous Data Requests with Message IDs](#asynchronous-data-requests-with-message-ids)

### `ui-action-error`

- a message that the iframe sends to the host to indicate that the action has failed. The original messageId is passed back to the host to allow the host to track the action. This is useful for `request-data` messages, but is not limited to this type.
- the payload is an object with the following properties:
  - `error` - the error that occurred

**Example:**
See [Asynchronous Data Requests with Message IDs](#asynchronous-data-requests-with-message-ids)

## Query Parameters

### `waitForRenderData`

- a query parameter that can be passed to the iframe to indicate that the iframe should wait for the render data to be passed before sending any messages
- the value of the query parameter is a boolean
- if the query parameter is present, the iframe will wait for the render data to be passed before sending any messages

**Example:**
See [Render Data](#render-data)

## Usage Examples

### Passing Render Data to the Iframe

### In the host:

```typescript
iframeSrc = "https://my-embeddable-ui.com?waitForRenderData=true";
iframe = document.createElement("iframe");
iframe.src = iframeSrc; // the iframe will wait for the render data to be passed before rendering
document.body.appendChild(iframe);

window.addEventListener("message", (event) => {
  if (event.data.type === "ui-lifecycle-iframe-ready") {
    iframe.contentWindow.postMessage(
      {
        type: "ui-lifecycle-iframe-render-data",
        payload: { renderData: { theme: "dark" } },
      },
      "*"
    );
  }
});
```

#### In the iframe:

```typescript
// In the iframe's script
const urlParams = new URLSearchParams(window.location.search);
if (urlParams.get("waitForRenderData") === "true") {
  let customRenderData = null;

  // The parent will send this message on load or when we notify it we're ready
  window.addEventListener("message", (event) => {
    // Add origin checks for security
    if (event.data.type === "ui-lifecycle-iframe-render-data") {
      // If the iframe has already received data, we don't need to do anything
      if (customRenderData) {
        return;
      } else {
        customRenderData = event.data.payload.renderData;
        // Now you can render the UI with the received data
        renderUI(renderData);
      }
    }
  });
  // We can let the parent know we're ready to receive data
  window.parent.postMessage({ type: "ui-lifecycle-iframe-ready" }, "*");
} else {
  // If the iframe doesn't need to wait for data, we can render the default UI immediately
  renderUI();
}
```

### Asynchronous Data Requests with Message IDs

### In the iframe:

```typescript
const requests = new Map<string, (response: any) => void>();

button.addEventListener("click", () => {
  const messageId = crypto.randomUUID();
  requests.set(messageId, (response) => {
    // handle the response
  });
  window.parent.postMessage(
    {
      type: "ui-request-data",
      messageId: "123",
      payload: {
        requestType: "get-payment-methods",
        params: {
          // any params needed for the request
        },
      },
    },
    "*"
  );
});

window.addEventListener("message", (event) => {
  if (event.data.type === "ui-action-response") {
    const { messageId, response } = event.data.payload;
    const callback = requests.get(messageId);
    if (callback) {
      callback(response);
      requests.delete(messageId);
    }
  }
});
```

### In the host:

```typescript
window.addEventListener("message", async (event) => {
  if (event.data.messageId) {
    iframe.contentWindow.postMessage(
      {
        type: "ui-action-received",
        messageId: event.data.messageId,
      },
      "*"
    );
  }

  if (event.data.type === "ui-request-data") {
    const {
      messageId,
      payload: { requestType, params },
    } = event.data;
    if (requestType === "get-payment-methods") {
      try {
        const paymentMethods = await fetchPaymentMethods(params);
        iframe.contentWindow.postMessage(
          {
            type: "ui-action-response",
            messageId: messageId,
            payload: { response: { paymentMethods } },
          },
          "*"
        );
      } catch (error) {
        iframe.contentWindow.postMessage(
          {
            type: "ui-action-error",
            messageId: messageId,
            payload: { error },
          },
          "*"
        );
      }
    }
  }
});
```

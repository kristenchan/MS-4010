---
lab:
    title: 'Introduction'
    module: 'LAB 02: Build your first action for declarative agents with API plugin by using Visual Studio Code'
---

# Introduction

Microsoft 365 Copilot agents let you create AI-powered assistants optimized for specific scenarios. Using instructions, you define the context for the agent and specify settings such as tone of voice or how it should respond. By configuring the agent's skills, you give it the ability to interact with external systems, trigger certain behavior under system conditions, or use custom workflow logic. One type of skill is actions that allow a declarative agent to communicate with APIs both for retrieving and modifying data.

![Diagram that shows the anatomy of a declarative agent for Microsoft 365 Copilot.](/media/LAB_02/1-anatomy-declarative-agent.png)

## Example scenario

Suppose you work in an organization, where you regularly order food from a local restaurant. The restaurant works with a daily menu which they publish on the Internet. You want to be able to quickly see what courses are available but also consider allergens in case you have guests. The restaurant exposes their menu via an API. Rather than building a separate app, you want to integrate the information into Microsoft 365 Copilot so that you can easily find the available dishes that you can order and find out their ingredients. You want to use natural language to browse through the menu and place orders.

## What will we be doing?

In this module, you build an action for a declarative agent with an API plugin. The action allows the agent to interact with an external system using its anonymous API. You learn to:

- **Create**: Create an API plugin that connects to an anonymous API.
- **Configure**: Configure the API plugin to show the data from the API.
- **Extend**: Extend a declarative agent with an action using an API plugin.
- **Provision**: Upload your declarative agent to Microsoft 365 Copilot and validate the results.

![Screenshot of a declarative agent that responds to a user with information from an external API.](/media/LAB_02/1-agent-response-api-plugin.png)

## Lab Duration

- **Estimated Time to complete**: 35 minutes

## Learning objectives

By the end of this module, you know how to integrate declarative agents with API plugins connected to anonymous APIs, to let them interact with external systems in real-time.


---
lab:
    title: 'Exercise 1 - Download project and examine files'
    module: 'LAB 02: Build your first action for declarative agents with API plugin by using Visual Studio Code'
---

# Exercise 1 - Download project and examine files

Extending a declarative agent with actions allows it to retrieve and update data stored in external systems in real-time. Using API plugins, you can connect to external systems through their APIs to retrieve and update information.

### Exercise Duration

- **Estimated Time to complete**: 10 minutes

## Task 1 - Download the starter project

Start by downloading the sample project. In a web browser:

1. Navigate to [https://github.com/microsoft/learn-declarative-agent-api-plugin-typescript](https://github.com/microsoft/learn-declarative-agent-api-plugin-typescript).
    1. Follow the steps to [download the repository source code](https://docs.github.com/repositories/working-with-files/using-files/downloading-source-code-archives#downloading-source-code-archives-from-the-repository-view) to your computer.
    1. Extract the contents of the downloaded ZIP file and extract it to your **Documents folder**.
    1. Open the folder in Visual Studio Code.

The sample project is a Teams Toolkit project that includes a declarative agent and an anonymous API running on Azure Functions. The declarative agent is identical to a newly created declarative agent using Teams Toolkit. The API belongs to a fictitious Italian restaurant and allows you to browse today's menu and place orders.

## Task 2 - Examine the API definition

First, look at the API definition of the Italian restaurant's API.

In Visual Studio Code:

1. In the **Explorer** view, open the **appPackage/apiSpecificationFile/ristorante.yml** file. The file is an OpenAPI specification that describes the API of the Italian restaurant.
1. Locate the **servers.url** property

    ```yaml
    servers:
      - url: http://localhost:7071/api
        description: Il Ristorante API server
    ```

    Notice that it's pointing to a local URL which matches the standard URL when running Azure Functions locally.

1. Locate the **paths** property, which contains two operations: **/dishes** for retrieving today's menu, and **/orders** for placing an order.

    > [!IMPORTANT]
    > Notice that each operation contains the **operationId** property that uniquely identifies the operation in the API specification. Copilot requires each operation to have a unique ID so that it knows which API it should call for specific user prompts.

## Task 3 - Examine the API implementation

Next, look at the sample API that you use in this exercise.

In Visual Studio Code:

1. In the **Explorer** view, open the **src/data.json** file. The file contains fictitious menu item for our Italian restaurant. Each dish consists of:

    - name,
    - description,
    - link to an image,
    - price,
    - in which course it's served,
    - type (dish or drink),
    - optionally a list of allergens

    In this exercise, APIs use this file as their data source.
1. Next, expand the **src/functions** folder. Notice two files named **dishes.ts** and **placeOrder.ts**. These files contain implementation of the two operations defined in the API specification.
1. Open the **src/functions/dishes.ts** file. Take a moment to review how the API is working. It starts with loading the sample data from the **src/functions/data.json** file.

    ```typescript
    import data from "../data.json";
    ```

    Next, it looks in the different query string parameters for possible filters that the client calling the API might pass.

    ```typescript
    const course = req.query.get('course');
    const allergensString = req.query.get('allergens');
    const allergens: string[] = allergensString ? allergensString.split(",") : [];
    const type = req.query.get('type');
    const name = req.query.get('name');
    ```

    Based on the filters specified on the request, the API filters the data set and returns a response.

1. Next, examine the API for placing orders defined in the **src/functions/placeOrder.ts** file. The API starts with referencing the sample data. Then, it defines the shape of the order that the client sends in the request body.

    ```typescript
    interface OrderedDish {
      name?: string;
      quantity?: number;
    }
    interface Order {
      dishes: OrderedDish[];
    }
    ```

    When the API processes the request, it first checks if the request contains a body and if it has the correct shape. If not, it rejects the request with a 400 Bad Request error.

    ```typescript
    let order: Order | undefined;
    try {
      order = await req.json() as Order | undefined;
    }
    catch (error) {
      return {
        status: 400,
        jsonBody: { message: "Invalid JSON format" },
      } as HttpResponseInit;
    }
    if (!order.dishes || !Array.isArray(order.dishes)) {
      return {
        status: 400,
        jsonBody: { message: "Invalid order format" }
      } as HttpResponseInit;
    }
    ```

    Next, the API resolves the request into dishes on the menu and calculates the total price.

    ```typescript
    let totalPrice = 0;
    const orderDetails = order.dishes.map(orderedDish => {
      const dish = data.find(d => d.name.toLowerCase().includes(orderedDish.name.toLowerCase()));
      if (dish) {
        totalPrice += dish.price * orderedDish.quantity;
        return {
          name: dish.name,
          quantity: orderedDish.quantity,
          price: dish.price,
        };
      }
      else {
        context.error(`Invalid dish: ${orderedDish.name}`);
        return null;
      }
    });
    ```

    > [!IMPORTANT]
    > Notice how the API expects the client to specify the dish by a part of its name rather than its ID. This is on purpose because large language models work better with words than numbers. Additionally, before calling the API to place the order, Copilot has the name of the dish readily available as part of the user's prompt. If Copilot had to refer to a dish by its ID, it would first need to retrieve which requires additional API requests and which Copilot can't do now.

    When the API is ready, it returns a response with a total price, and a made-up order ID, and status.

    ```typescript
    const orderId = Math.floor(Math.random() * 10000);
    return {
      status: 201,
      jsonBody: {
        order_id: orderId,
        status: "confirmed",
        total_price: totalPrice,
      }
    } as HttpResponseInit;
    ```


---
lab:
    title: 'Exercise 2 - Build API plugin definition'
    module: 'LAB 02: Build your first action for declarative agents with API plugin by using Visual Studio Code'
---

# Exercise 2 - Build API plugin definition

The next step is to add the plugin definition to the project. The plugin definition contains the following information:

- What actions the plugin can perform.
- What's the shape of the data that it expects and returns.
- How the declarative agent must call the underlying API.

### Exercise Duration

- **Estimated Time to complete**: 10 minutes

## Task 1 - Add the basic plugin definition structure

In Visual Studio Code:

1. In the **appPackage** folder, add a new file named **ai-plugin.json**.
1. Paste the following contents:

    ```json
    {
      "$schema": "https://developer.microsoft.com/json-schemas/copilot/plugin/v2.1/schema.json",
      "schema_version": "v2.1",
      "namespace": "ilristorante",
      "name_for_human": "Il Ristorante",
      "description_for_human": "See the today's menu and place orders",
      "description_for_model": "Plugin for getting the today's menu, optionally filtered by course and allergens, and placing orders",
      "functions": [ 
      ],
      "runtimes": [
      ],
      "capabilities": {
        "localization": {},
        "conversation_starters": []
      }
    }
    ```

    The file contains a basic structure for an API plugin with a description for the human and the model. The **description_for_model** includes detailed information about what the plugin can do to help the agent understand when it should consider invoking it.
1. Save your changes.

## Task 2 - Define functions

An API plugin defines one or more functions that map to API operations defined in the API specification. Each function consists of a name and a description and a response definition that instructs the agent how to display the data to users.

### Define a function to retrieve the menu

Start with defining a function to retrieve the information about today's menu.

In Visual Studio Code:

1. Open the **appPackage/ai-plugin.json** file.
1. In the **functions** array, add the following snippet:

    ```json
    {
      "name": "getDishes",
      "description": "Returns information about the dishes on the menu. Can filter by course (breakfast, lunch or dinner), name, allergens, or type (dish, drink).",
      "capabilities": {
        "response_semantics": {
          "data_path": "$.dishes",
          "properties": {
            "title": "$.name",
            "subtitle": "$.description"
          }
        }
      }
    }
    ```

    You start by defining a function that invokes the **getDishes** operation from the API specification. Next, you provide a function description. This description is important because Copilot uses it to decide which function to invoke for a user's prompt.

    In the response_semantics property, you specify how Copilot should display the data it receives from the API. Because the API returns the information about the dishes on the menu in the **dishes** property, you set the **data_path** property to the `$.dishes` JSONPath expression.

    Next, in the **properties** section, you map which properties from the API response represent the title, description, and URL. Because in this case the dishes don't have a URL, you only map the **title** and **description**.

1. The complete code snippet looks like:

    ```json
    {
      "$schema": "https://developer.microsoft.com/json-schemas/copilot/plugin/v2.1/schema.json",
      "schema_version": "v2.1",
      "namespace": "ilristorante",
      "name_for_human": "Il Ristorante",
      "description_for_human": "See the today's menu and place orders",
      "description_for_model": "Plugin for getting the today's menu, optionally filtered by course and allergens, and placing orders",
      "functions": [
        {
          "name": "getDishes",
          "description": "Returns information about the dishes on the menu. Can filter by course (breakfast, lunch or dinner), name, allergens, or type (dish, drink).",
          "capabilities": {
            "response_semantics": {
              "data_path": "$.dishes",
              "properties": {
                "title": "$.name",
                "subtitle": "$.description"
              }
            }
          }
        }
      ],
      "runtimes": [
      ],
      "capabilities": {
        "localization": {},
        "conversation_starters": []
      }
    }
    ```

1. Save your changes.

### Define a function to place the order

Next, define a function to place the order.

In Visual Studio Code:

1. Open the **appPackage/ai-plugin.json** file.
1. To the end of the **functions** array, add the following snippet:

    ```json
    {
      "name": "placeOrder",
      "description": "Places an order and returns the order details",
      "capabilities": {
        "response_semantics": {
          "data_path": "$",
          "properties": {
            "title": "$.order_id",
            "subtitle": "$.total_price"
          }
        }
      }
    }
    ```

    You start by referring to the API operation with ID **placeOrder**. Then, you provide a description that Copilot uses to match this function against a user's prompt. Next, you instruct Copilot how to return the data. Following is the data that the API returns after placing an order:

    ```json
    {
      "order_id": 6532,
      "status": "confirmed",
      "total_price": 21.97
    }
    ```

    Because the data that you want to show is located directly in the root of the response object, you set the **data_path** to **$** which indicates the top node of the JSON object. You define the title to show the order's number and the subtitle its price.

1. The complete file looks like:

    ```json
    {
      "$schema": "https://developer.microsoft.com/json-schemas/copilot/plugin/v2.1/schema.json",
      "schema_version": "v2.1",
      "namespace": "ilristorante",
      "name_for_human": "Il Ristorante",
      "description_for_human": "See the today's menu and place orders",
      "description_for_model": "Plugin for getting the today's menu, optionally filtered by course and allergens, and placing orders",
      "functions": [
        {
          "name": "getDishes",
          "description": "Returns information about the dishes on the menu. Can filter by course (breakfast, lunch or dinner), name, allergens, or type (dish, drink).",
          ...trimmed for brevity
        },
        {
          "name": "placeOrder",
          "description": "Places an order and returns the order details",
          "capabilities": {
            "response_semantics": {
              "data_path": "$",
              "properties": {
                "title": "$.order_id",
                "subtitle": "$.total_price"
              }
            }
          }
        }
      ],
      "runtimes": [
      ],
      "capabilities": {
        "localization": {},
        "conversation_starters": []
      }
    }
    ```

1. Save your changes.

## Task 3 - Define runtimes

After defining functions for Copilot to invoke, the next step is to instruct it how it should call them. You do that in the **runtimes** section of the plugin definition.

In Visual Studio Code:

1. Open the **appPackage/ai-plugin.json** file.
1. In the runtimes array, add the following code:

    ```json
    {
      "type": "OpenApi",
      "auth": {
        "type": "None"
      },
      "spec": {
        "url": "apiSpecificationFile/ristorante.yml"
      },
      "run_for_functions": [
        "getDishes",
        "placeOrder"
      ]
    }
    ```

    You start with instructing Copilot that you provide it with OpenAPI information about the API (**type: OpenApi**) to call and that it's anonymous (**auth.type: None**). Next, in the **spec** section, you specify the relative path to the API specification located in your project. Finally, in the **run_for_functions** property, you list all functions that belong to this API.

1. The complete file looks like:

    ```json
    {
      "$schema": "https://developer.microsoft.com/json-schemas/copilot/plugin/v2.1/schema.json",
      "schema_version": "v2.1",
      "namespace": "ilristorante",
      "name_for_human": "Il Ristorante",
      "description_for_human": "See the today's menu and place orders",
      "description_for_model": "Plugin for getting the today's menu, optionally filtered by course and allergens, and placing orders",
      "functions": [
        {
          "name": "getDishes",
          ...trimmed for brevity
        },
        {
          "name": "placeOrder",
          ...trimmed for brevity
        }
      ],
      "runtimes": [
        {
          "type": "OpenApi",
          "auth": {
            "type": "None"
          },
          "spec": {
            "url": "apiSpecificationFile/ristorante.yml"
          },
          "run_for_functions": [
            "getDishes",
            "placeOrder"
          ]
        }
      ],
      "capabilities": {
        "localization": {},
        "conversation_starters": []
      }
    }
    ```

1. Save your changes.


---
lab:
    title: 'Exercise 3 - Connect the plugin definition to the declarative agent'
    module: 'LAB 02: Build your first action for declarative agents with API plugin by using Visual Studio Code'
---

# Exercise 3 - Connect the plugin definition to the declarative agent

After you complete building the API plugin definition, the next step is to register it with the declarative agent. When users interact with the declarative agent, it matches the user's prompt against the defined API plugins and invokes the relevant functions.

### Exercise Duration

- **Estimated Time to complete**: 5 minutes

## Task 1 - Connect the plugin definition to the declarative agent

In Visual Studio Code:

1. Open the **appPackage/declarativeAgent.json** file.
1. After the **instructions** property, add the following code snippet:

    ```json
    "actions": [
      {
        "id": "menuPlugin",
        "file": "ai-plugin.json"
      }
    ]
    ```

    Using this snippet, you connect the declarative agent to the API plugin. You specify a unique ID for the plugin and instruct the agent where it can find the plugin's definition.

1. The complete file looks like:

    ```json
    {
      "$schema": "https://developer.microsoft.com/json-schemas/copilot/declarative-agent/v1.0/schema.json",
      "version": "v1.0",
      "name": "Declarative agent",
      "description": "Declarative agent created with Teams Toolkit",
      "instructions": "$[file('instruction.txt')]",
      "actions": [
        {
          "id": "menuPlugin",
          "file": "ai-plugin.json"
        }
      ]
    }
    ```

1. Save your changes.

## Task 2 - Update declarative agent information and instruction

The declarative agent that you're building in this exercise helps users browse the menu of the local Italian restaurant and place orders. To optimize the agent for this scenario, update its name, description, and instructions.

In Visual Studio Code:

1. Update the declarative agent information:
    1. Open the **appPackage/declarativeAgent.json** file.
    1. Update the value of the **name** property to **Il Ristorante**.
    1. Update the value of the **description** property to **Order the most delicious Italian dishes and drinks from the comfort of your desk.**
    1. Save the changes.
1. Update declarative agent's instructions:
    1. Open the **appPackage/instruction.txt** file.
    1. Replace its contents with:

        ```markdown
        You are an assistant specialized in helping users explore the menu of an Italian restaurant and place orders. You interact with the restaurant's menu API and guide users through the ordering process, ensuring a smooth and delightful experience. Follow the steps below to assist users in selecting their desired dishes and completing their orders:
        
        ### General Behavior:
        - Always greet the user warmly and offer assistance in exploring the menu or placing an order.
        - Use clear, concise language with a friendly tone that aligns with the atmosphere of a high-quality local Italian restaurant.
        - If the user is browsing the menu, offer suggestions based on the course they are interested in (breakfast, lunch, or dinner).
        - Ensure the conversation remains focused on helping the user find the information they need and completing the order.
        - Be proactive but never pushy. Offer suggestions and be informative, especially if the user seems uncertain.
        
        ### Menu Exploration:
        - When a user requests to see the menu, use the `GET /dishes` API to retrieve the list of available dishes, optionally filtered by course (breakfast, lunch, or dinner).
          - Example: If a user asks for breakfast options, use the `GET /dishes?course=breakfast` to return only breakfast dishes.
        - Present the dishes to the user with the following details:
          - Name of the dish
          - A tasty description of the dish
          - Price in € (Euro) formatted as a decimal number with two decimal places
          - Allergen information (if relevant)
          - Don't include the URL.
        
        ### Beverage Suggestion:
        - If the order does not already include a beverage, suggest a suitable beverage option based on the course.
        - Use the `GET /dishes?course={course}&type=drink` API to retrieve available drinks for that course.
        - Politely offer the suggestion: *"Would you like to add a beverage to your order? I recommend [beverage] for [course]."*
        
        ### Placing the Order:
        - Once the user has finalized their order, use the `POST /order` API to submit the order.
          - Ensure the request includes the correct dish names and quantities as per the user's selection.
          - Example API payload:
         
            ```json
            {
              "dishes": [
                {
                  "name": "frittata",
                  "quantity": 2
                },
                {
                  "name": "cappuccino",
                  "quantity": 1
                }
              ]
            }
            ```
        
        ### Error Handling:
        - If the user selects a dish that is unavailable or provides an invalid dish name, respond gracefully and suggest alternative options.
          - Example: *"It seems that dish is currently unavailable. How about trying [alternative dish]?"*
        - Ensure that any errors from the API are communicated politely to the user, offering to retry or explore other options.
        ```

        Notice that in the instructions, we define the general behavior of the agent and instruct it what it's capable of. We also include instructions for specific behavior around placing an order, including the shape of the data that the API expects. We include this information to ensure that the agent works as intended.

    > [!NOTE]
    > To preserve formatting, you may need to perform multiple copy/paste operations into Notepad before copying into Visual Studio Code.

    1. Save the changes.

1. To help users, understand what they can use the agent for, add conversation starters:
    1. Open the **appPackage/declarativeAgent.json** file.
    1. After the **instructions** property, add a new property named **conversation_starters**:

        ```json
        "conversation_starters": [
          {
            "text": "What's for lunch today?"
          },
          {
            "text": "What can I order for dinner that is gluten-free?"
          }
        ]
        ```

    1. The complete file looks like:

        ```json
        {
          "$schema": "https://developer.microsoft.com/json-schemas/copilot/declarative-agent/v1.0/schema.json",
          "version": "v1.0",
          "name": "Il Ristorante",
          "description": "Order the most delicious Italian dishes and drinks from the comfort of your desk.",
          "instructions": "$[file('instruction.txt')]",
          "conversation_starters": [
            {
              "text": "What's for lunch today?"
            },
            {
              "text": "What can I order for dinner that is gluten-free?"
            }
          ],
          "actions": [
            {
              "id": "menuPlugin",
              "file": "ai-plugin.json"
            }
          ]
        }
        ```

    1. Save your changes.

## Task 3 - Update the API URL

Before you can test your declarative agent, you need to update the URL of the API in the API specification file. Right now, the URL is set to `http://localhost:7071/api` which is the URL that Azure Functions uses when running locally. However, because you want Copilot to call your API from the cloud, you need to expose your API to the internet. Teams Toolkit automatically exposes your local API over the internet, by creating a dev tunnel. Each time you start debugging your project, Teams Toolkit starts a new dev tunnel and stores its URL in the **OPENAPI_SERVER_URL** variable. You can see how Teams Toolkit starts the tunnel and stores its URL in the **.vscode/tasks.json** file, in the **Start local tunnel** task:

```json
{
  // Start the local tunnel service to forward public URL to local port and inspect traffic.
  // See https://aka.ms/teamsfx-tasks/local-tunnel for the detailed args definitions.
  "label": "Start local tunnel",
  "type": "teamsfx",
  "command": "debug-start-local-tunnel",
  "args": {
    "type": "dev-tunnel",
    "ports": [
      {
        "portNumber": 7071,
        "protocol": "http",
        "access": "public",
        "writeToEnvironmentFile": {
          "endpoint": "OPENAPI_SERVER_URL", // output tunnel endpoint as OPENAPI_SERVER_URL
        }
      }
    ],
    "env": "local"
  },
  "isBackground": true,
  "problemMatcher": "$teamsfx-local-tunnel-watch"
}
```

To use this tunnel, you need to update your API specification to use the **OPENAPI_SERVER_URL** variable.

In Visual Studio Code:

1. Open the **appPackage/apiSpecificationFile/ristorante.yml** file.
1. Change the value of the **servers.url** property to **${{OPENAPI_SERVER_URL}}/api**.
1. The changed file looks like:

    ```yaml
    openapi: 3.0.0
    info:
      title: Il Ristorante menu API
      version: 1.0.0
      description: API to retrieve dishes and place orders for Il Ristorante.
    servers:
      - url: ${{OPENAPI_SERVER_URL}}/api
        description: Il Ristorante API server
    paths:
      ...trimmed for brevity
    ```

1. Save your changes.

Your API plugin is finished and integrated with a declarative agent. Continue with testing the agent in Microsoft 365 Copilot.



---
lab:
  title: 'Exercise 4 - Test the declarative agent with API plugin in Microsoft 365 Copilot'
  module: 'LAB 02: Build your first action for declarative agents with API plugin by using Visual Studio Code'
---

# Exercise 4 - Test the declarative agent with API plugin in Microsoft 365 Copilot

Extending a declarative agent with actions allows it to retrieve and update data stored in external systems in real-time. Using API plugins, you can connect to external systems through their APIs to retrieve and update information.

### Exercise Duration

- **Estimated Time to complete**: 10 minutes

## Task 1 - Test the declarative agent

The final step is to test the declarative agent with API plugin in Microsoft 365 Copilot.

In Visual Studio Code:

1. From the **Activity Bar**, choose **Teams Toolkit**.
1. In the **Accounts** section, ensure that you're signed in to your Microsoft 365 tenant with Microsoft 365 Copilot.

  ![Screenshot of the Teams Toolkit accounts section in Visual Studio Code.](/media/LAB_02/3-teams-toolkit-accounts.png)

1. From the **Activity Bar**, choose **Run and Debug**.
1. Select the **Debug in Copilot** configuration and start debugging using the **Start Debugging** button.  

  ![Screenshot of the Debug in Copilot configuration in Visual Studio Code.](/media/LAB_02/3-visual-studio-code-start-debugging.png)

1. Visual Studio Code builds and deploys your project to your Microsoft 365 tenant and opens a new web browser window.

In the web browser:

1. When prompted, sign in with the account that belongs to your Microsoft 365 tenant with Microsoft 365 Copilot.
1. From the side bar, select **Il Ristorante**.

  ![Screenshot of the Microsoft 365 Copilot interface with the Il Ristorante agent selected.](/media/LAB_02/3-copilot-select-agent.png)

1. Choose the **What's for lunch today?** conversation starter and submit the prompt.

  ![Screenshot of the Microsoft 365 Copilot interface with the lunch prompt.](/media/LAB_02/3-copilot-lunch-prompt.png)

1. When prompted, examine the data that the agent sends to the API and confirm using the **Allow once** button.

  ![Screenshot of the Microsoft 365 Copilot interface with the lunch confirmation.](/media/LAB_02/3-copilot-lunch-confirm.png)

1. Wait for the agent to respond. Notice that while it shows citations for the information it retrieves from the API, the popup only shows the dish's title. It doesn't show any additional information, because the API plugin doesn't define an Adaptive Card template.

  ![Screenshot of the Microsoft 365 Copilot interface with the lunch response.](/media/LAB_02/3-copilot-lunch-response.png)

1. Place an order, by typing in the prompt text box: **1x spaghetti, 1x iced tea** and submit the prompt.
1. Examine the data that the agent sends to the API and continue using the **Confirm** button.

  ![Screenshot of the Microsoft 365 Copilot interface with the order confirmation.](/media/LAB_02/3-copilot-order-confirm.png)

1. Wait for the agent to place the order and return the order summary. Once again, notice that the agent shows the order summary in plain text because it doesn't have an Adaptive Card template.

  ![Screenshot of the Microsoft 365 Copilot interface with the order response.](/media/LAB_02/3-copilot-order-response.png)

1. Go back to Visual Studio Code and stop debugging.
1. Switch to the **Terminal** tab and close all active terminals.

  ![Screenshot of the Visual Studio Code terminal tab with the option to close all terminals.](/media/LAB_02/3-visual-studio-code-close-terminal.png)

You're looking into building a solution for your organization that allows you to order food from a local restaurant. The solution must use natural language and integrate with Microsoft 365 Copilot.



You did some research and found that building a declarative agent for Microsoft 365 Copilot, integrated with an API plugin is suitable for your needs, because:

- It allows you to build an AI-powered assistant that understands natural language.
- It allows you to build on top of the infrastructure of Microsoft 365 Copilot.
- It allows you to use instructions to optimize the assistant to your scenario.
- It allows you to connect the agent to the restaurant's API to browse their menu and place orders.

Previously, you and your colleagues had to spend valuable time on the phone to learn about today's menu and place an order that meets everyone's dietary needs.

By building a declarative agent, you and your colleagues can quickly find out today's menu and filter it according to their preferences. Having the agent available in Microsoft 365 Copilot is convenient because it's available at your fingertips where you already are without having to navigate to a separate application.




























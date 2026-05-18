---
title: |
  Tutorial: Build a TypeScript app with the Azure Cosmos DB emulator and deploy to Azure
description: In this tutorial, use the Azure Cosmos DB Linux emulator for local development, build a TypeScript app, and deploy to Azure Cosmos DB.
ms.topic: tutorial
ms.date: 04/15/2026
ai-usage: ai-generated
# Customer intent: As a developer, I want to build and test an app locally using the Azure Cosmos DB emulator so that I can migrate to Azure Cosmos DB with minimal friction.
---

# Tutorial: Build a TypeScript app with the Azure Cosmos DB emulator and deploy to Azure Cosmos DB

In this tutorial, you use the Azure Cosmos DB Linux emulator as a local development database, build a TypeScript application, and then migrate to Azure Cosmos DB by swapping your credentials.

In this tutorial, you:

> [!div class="checklist"]
> - Start the Azure Cosmos DB emulator in a Docker container
> - Create a TypeScript application
> - Connect the application to the local emulator
> - Create an Azure Cosmos DB account
> - Deploy the application to Azure Cosmos DB

## Prerequisites

- [Node.js 22 or newer](https://nodejs.org/en/download/)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [Azure CLI](/cli/azure/install-azure-cli)
- An Azure subscription. If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/pricing/purchase-options/azure-account?cid=msft_learn) before you begin.

## Start the emulator

Use the Docker container image for the Linux-based Azure Cosmos DB emulator to create a local development database.

1. Pull the emulator container image.

    ```bash
    docker pull mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator:vnext-preview
    ```

1. Start the emulator container.

    ```bash
    docker run --detach --publish 8081:8081 --publish 1234:1234 --name cosmos-db mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator:vnext-preview
    ```

1. Verify the container is running.

    ```bash
    docker ps --filter "name=cosmos-db"
    ```

    > [!IMPORTANT]
    > The emulator gateway endpoint is available at `https://localhost:8081` and the data explorer is available at `http://localhost:1234`. The emulator uses a well-known authentication key:
    >
    > ```connection-string
    > AccountEndpoint=http://localhost:8081/;AccountKey=C2y6yDjf5/R+ob0N8A7Cgv30VRDJIWEHLM+4QDU5DE2nQ9nDuVTqobD4b8mGGyPMbIZnqyMsEcaGQy67XIw/Jw==;
    > ```
    >
    > | Setting | Value |
    > | --- | --- |
    > | **Endpoint** | `https://localhost:8081` |
    > | **Key** | `C2y6yDjf5/R+ob0N8A7Cgv30VRDJIWEHLM+4QDU5DE2nQ9nDuVTqobD4b8mGGyPMbIZnqyMsEcaGQy67XIw/Jw==` |
    >

## Create the application

Create a Next.js application and connect it to the local emulator.

1. Create a new Next.js application in an empty folder. Accept any default settings.

    ```bash
    npx create-next-app .
    ```

1. Install the Node.js client library for Azure Cosmos DB.

    ```bash
    npm install --save @azure/cosmos
    ```

1. Create a new file named `app/data.tsx` with the following code. This file contains the server actions that interact with the database.

    ```typescript
    "use server";

    import { CosmosClient } from "@azure/cosmos";
    import { revalidatePath } from "next/cache";

    const connectionString = "<cosmos-db-connection-string>";

    const client = new CosmosClient(connectionString);

    const { database } = await client.databases.createIfNotExists({ id: "work-management" });
    const { container } = await database.containers.createIfNotExists({ id: "tasks", partitionKey: "/id" });

    export type Todo = {
      id: string;
      title: string;
      completed: boolean;
    };

    export async function getTodos(): Promise<Todo[]> {
      const { resources } = await container.items
        .query("SELECT * FROM c ORDER BY c._ts DESC")
        .fetchAll();
      return resources.map((doc) => ({
        id: doc.id,
        title: doc.title,
        completed: doc.completed,
      }));
    }

    export async function addTodo(formData: FormData) {
      const title = formData.get("title") as string;
      if (!title?.trim()) return;
      await container.items.create({
        title: title.trim(),
        completed: false,
      });
      revalidatePath("/");
    }

    export async function toggleTodo(formData: FormData) {
      const id = formData.get("id") as string;
      const { resource: doc } = await container.item(id, id).read();
      if (doc) {
        await container.item(id, id).replace({
          ...doc,
          completed: !doc.completed,
        });
      }
      revalidatePath("/");
    }

    export async function deleteTodo(formData: FormData) {
      const id = formData.get("id") as string;
      await container.item(id, id).delete();
      revalidatePath("/");
    }
    ```

    > [!IMPORTANT]
    > Replace the connection string placeholder with the credentials specified earlier in this tutorial when starting the container.

1. Delete the existing `app/page.tsx` file and replace it with the following code.

    ```react
    import { getTodos, addTodo, toggleTodo, deleteTodo } from "./data";

    export default async function Home() {
      const todos = await getTodos();

      return (
        <div className="min-h-screen bg-black text-zinc-50 font-sans">
          <main className="max-w-lg mx-auto py-16 px-8">
            <h1 className="text-2xl font-semibold tracking-tight mb-8">Todo App</h1>

            <form action={addTodo} className="flex gap-2 mb-8">
              <input
                name="title"
                placeholder="Add a todo..."
                required
                className="flex-1 rounded-md border border-zinc-700 bg-zinc-900 px-3 py-2 text-sm text-zinc-100 placeholder:text-zinc-500 focus:outline-none focus:ring-2 focus:ring-zinc-600"
              />
              <button type="submit" className="rounded-md bg-zinc-100 px-4 py-2 text-sm font-medium text-zinc-900 hover:bg-zinc-300 transition-colors">
                Add
              </button>
            </form>

            <ul className="space-y-2">
              {todos.map((todo) => (
                <li key={todo.id} className="flex items-center gap-3 rounded-md border border-zinc-800 bg-zinc-900 px-3 py-2">
                  <form action={toggleTodo}>
                    <input type="hidden" name="id" value={todo.id} />
                    <button type="submit" className="text-lg leading-none">{todo.completed ? "✅" : "⬜"}</button>
                  </form>
                  <span className={`flex-1 text-sm ${todo.completed ? "line-through text-zinc-500" : "text-zinc-100"}`}>{todo.title}</span>
                  <form action={deleteTodo}>
                    <input type="hidden" name="id" value={todo.id} />
                    <button type="submit" className="text-sm text-red-400 hover:text-red-300 transition-colors">✕</button>
                  </form>
                </li>
              ))}
            </ul>

            {todos.length === 0 && <p className="text-center text-sm text-zinc-500 mt-4">No todos yet.</p>}
          </main>
        </div>
      );
    }
    ```

1. Run the application locally.

    ```bash
    npm run dev
    ```

1. Open `http://localhost:3000` in your browser to verify the application is running and connected to the emulator.

    :::image type="content" source="media/development-loop/running-app.png" lightbox="media/development-loop/running-app-full.png" alt-text="Screenshot of the new application running in a web browser with multiple items persisted to the database.":::

    > [!TIP]
    > You can review the data this application creates in the Azure Cosmos DB Emulator Data Explorer at `http://localhost:1234`. You can also use the  [Azure Cosmos DB extension for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-cosmosdb) The database is named `work-management` and the container is named `tasks`.

1. Stop the application.

## Create an Azure Cosmos DB account

Create an Azure Cosmos DB account to host your application data in the cloud.

1. Create variables for the account name and target resource group name.

    ```azurecli
    RESOURCE_GROUP_NAME="<resource-group-name>"
    ACCOUNT_NAME="<account-name>"
    ```

    > [!IMPORTANT]
    > Replace `<resource-group-name>` and `<account-name>` with your own values. The account name must be globally unique.

1. Create an Azure Cosmos DB account.

    ```azurecli
    az cosmosdb create \
      --resource-group $RESOURCE_GROUP_NAME \
      --name $ACCOUNT_NAME
    ```

1. Get the connection string for the account.

    ```azurecli
    az cosmosdb keys list \
      --resource-group $RESOURCE_GROUP_NAME \
      --name $ACCOUNT_NAME \
      --type connection-strings \
      --query "connectionStrings[0].connectionString" \
      --output tsv
    ```

1. Record the connection string value. You use it in the next step.

## Connect the application to Azure Cosmos DB

Update the application to connect to Azure Cosmos DB instead of the local emulator.

1. In `app/data.tsx`, replace the emulator connection string with the Azure Cosmos DB connection string.

    ```typescript
    const connectionString = "<azure-cosmos-db-connection-string>";
    ```

    > [!IMPORTANT]
    > Replace `<azure-cosmos-db-connection-string>` with the connection string you recorded in the previous step.

1. Start the application again.

    ```bash
    npm run dev
    ```

1. Open `http://localhost:3000` in your browser. The application now reads and writes data to your Azure Cosmos DB account.

## Clean up resources

When you no longer need the resources created in this tutorial, delete them to avoid incurring charges.

1. Delete the Azure Cosmos DB account.

    ```azurecli
    az cosmosdb delete \
      --resource-group $RESOURCE_GROUP_NAME \
      --name $ACCOUNT_NAME \
      --yes
    ```

1. Stop and remove the emulator container.

    ```bash
    docker stop cosmos-db
    docker rm cosmos-db
    ```

## Next step

> [!div class="nextstepaction"]
> [Quickstart: Azure Cosmos DB with Node.js](/azure/cosmos-db/quickstart-nodejs)

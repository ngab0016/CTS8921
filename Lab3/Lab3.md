# CST8921 – Cloud Industry Trends
## Lab 3 – Cosmos DB Change Feed
### Lab Report

**Student Number:** 041196196
**Date:** 22/02/2025

---

## Part 1: Azure Cosmos DB Setup

### Screenshot 1 – Cosmos DB Account Overview
*Cosmos DB account overview page after deployment*

![alt text](<Screenshot 2026-02-21 at 13.51.36.png>)

---

### Screenshot 2 – Keys Pane
*Keys pane showing the endpoint and primary key*

![alt text](<Screenshot 2026-02-21 at 13.56.28.png>)

---

### Screenshot 3 – Data Explorer with Containers
*Data Explorer showing both `products` and `productslease` containers under the `cosmicworks` database*

![alt text](<Screenshot 2026-02-21 at 14.03.51.png>)

---

## Part 2: Visual Studio Code Configuration

### Screenshot 4 – Edited script.cs
*VS Code showing the edited `script.cs` file with endpoint and key configured*

![alt text](<Screenshot 2026-02-21 at 14.10.40.png>)

---

## Part 3: Change Feed Processor

### Screenshot 5 – App Running
*Terminal showing the app running and "Listening for changes..."*...Shown below the errors

![alt text](<Screenshot 2026-02-21 at 14.20.53.png>)

---

### Screenshot 6 – Detected Operations in Terminal
*VS Code terminal showing "Detected Operation" messages for products added via Azure Portal Data Explorer*

![alt text](<Screenshot 2026-02-21 at 14.53.17.png>)

---

### Screenshot 7 – Products Container Items
*Data Explorer showing items in the `products` container*

![alt text](<Screenshot 2026-02-21 at 17.02.11.png>)

---


## Part 4: Azure Function App with Cosmos DB Trigger

### Screenshot 8 – Function App Overview
*Azure Portal showing the Function App overview with status "Running"*

![alt text](<Screenshot 2026-02-21 at 15.32.20.png>)

---

### Screenshot 9 – ItemsListener Invocations
*Azure Portal showing the ItemsListener Invocations page with successful invocations*

![alt text](<Screenshot 2026-02-21 at 17.26.38.png>)

---

### Screenshot 10 – Invocation Details
*Azure Portal showing the details of a successful invocation with "Detected Operation" log output*

![alt text](<Screenshot 2026-02-21 at 17.26.38-1.png>)

---

## Part 5: Cleanup

### Screenshot 11 – Resource Group Deletion
*Azure Portal showing the resource group deletion confirmation screen*

![alt text](<Screenshot 2026-02-21 at 17.42.30.png>)

---

## Summary

In this lab, I explored the capabilities of Azure Cosmos DB change feed using the Azure Portal, Azure CLI, and Visual Studio Code. I:

- Created an Azure Cosmos DB for NoSQL account with serverless capacity
- Created the `cosmicworks` database with `products` and `productslease` containers
- Configured and ran a .NET change feed processor application that detected create/update operations on the `products` container
- Created an Azure Function App with a Cosmos DB trigger (`ItemsListener`) that automatically fires when changes are detected in the change feed
- Verified successful function invocations via the Azure Portal Monitor/Invocations pane
- Deleted all resources after completing the lab
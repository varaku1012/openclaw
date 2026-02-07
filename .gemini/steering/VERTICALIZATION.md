# Verticalization Strategy

"Verticalization" in OpenClaw means tailoring the engine for a specific industry (e.g., Restaurants, Marketing, Auto Dealerships). This is achieved by combining **Extensions** (connectivity) and **Skills** (intelligence).

## The Recipe for a Vertical

To create a Vertical (e.g., "OpenClaw for Restaurants"), you typically need:

1.  **A Custom Channel (Optional):** If the vertical requires a specific communication interface (e.g., a "Table Reservation Widget" on a website), build it as an **Extension**.
2.  **Domain Skills:** A set of tools and prompts that understand the business logic (e.g., "Check Menu", "Book Table").
3.  **Specialized Agent Config:** A system prompt that defines the persona (e.g., "You are a helpful Maître D'").

## Step 1: Create an Extension (Connectivity)
Use Extensions to connect OpenClaw to industry-specific software or communication channels.

*   **Location:** `extensions/<vertical-name>`
*   **Use Case:**
    *   Connecting to a Restaurant POS API.
    *   Connecting to a CRM (HubSpot, Salesforce) for Marketing.
    *   Connecting to a Dealer Management System (DMS) for Auto.
*   **How-To:**
    *   Copy a simple extension like `extensions/googlechat` or use the `skill-creator` to scaffold.
    *   Implement `OpenClawPluginApi` from `src/plugin-sdk`.
    *   Define the `openclaw.plugin.json` metadata.

## Step 2: Create a Skill (Intelligence)
Use Skills to give the Agent the tools it needs to perform tasks in that vertical.

*   **Location:** `skills/<skill-name>`
*   **Use Case:**
    *   `skills/food-order`: Tools to browse menus and place orders.
    *   `skills/crm-lookup`: Tools to find customer details.
    *   `skills/inventory-check`: Tools to check car availability.
*   **How-To:**
    *   Create a folder in `skills/`.
    *   Define the tools (using scripts or native code).
    *   Provide a `SKILL.md` description so the Agent knows when to use it.

## Step 3: Deployment & Configuration
*   **Bundle:** Package the Extension and Skill with the core OpenClaw engine.
*   **Configure:** Set up the Agent's system prompt to enforce the role.
    *   *Example:* "You are an assistant for 'Luigi's Trattoria'. specialized in taking reservations and answering menu questions. Always check availability before confirming."

## Example: Restaurant Vertical
1.  **Extension:** `extensions/restaurant-pos`
    *   Connects to the restaurant's Point-of-Sale system API.
    *   Provides real-time table status.
2.  **Skill:** `skills/menu-knowledge`
    *   Contains the full menu, ingredients, and allergen info.
    *   Tool: `search_menu(query: string)`
3.  **Channel:** `extensions/web-widget`
    *   A lightweight chat widget embedded on the restaurant's website.

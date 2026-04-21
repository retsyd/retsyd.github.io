---
title: "FormulAI: Using LangChain to Generate Skincare Formulations"
date: 2023-06-11
tags: ["llm", "langchain", "python", "streamlit", "hackathon"]
summary: "A hackathon project that chains LLM calls with a product ingredient database and Wikipedia to generate skincare formulations — ingredients, assembly protocols, and allergen warnings."
---

This was a weekend hackathon project built with three teammates. The idea: given a prompt like "a moisturiser with hyaluronic acid," generate a complete formulation, ingredient list, assembly protocol, and allergen warnings, by chaining together an LLM, a product ingredient database, and Wikipedia.

The result was [FormulAI](https://github.com/blue-moon22/formulAI), a Streamlit app built on LangChain and GPT-3.5.

## How It Works

The app takes a text prompt (e.g. "a cream with retinol") and a formulation type (cream, spray, or any), then runs four chained steps:

1. **CSV Agent**: A LangChain CSV agent queries a dataset of ~200 real skincare products with their full ingredient lists, scraped and cleaned from product labels. The agent finds products with similar ingredients to the request, giving the LLM real-world formulation context rather than generating from pure parametric knowledge.

2. **Ingredients Chain**: An LLM chain takes the user's requested chemical, the CSV agent's findings, and the selected formulation type, then generates a full ingredient list with each ingredient's function (humectant, emulsifier, preservative, etc.). The prompt tells the model to leverage the real formulations from the dataset and suggest similar chemicals if the exact one isn't present.

3. **Protocol Chain**: A second LLM chain takes the generated ingredients plus Wikipedia research on the requested chemical and writes an assembly protocol — the order of operations for combining the ingredients.

4. **Allergen Chain**: A final chain takes the ingredient list and identifies potential allergens.

Each chain has its own `ConversationBufferMemory`, so the conversation history is preserved and visible in the sidebar for debugging. The Wikipedia lookup runs via LangChain's `WikipediaAPIWrapper` to pull in reference material on the target chemical.

## The Dataset

The ingredient database (`skincare_products_clean.csv`) contains real product formulations across moisturisers, serums, oils, mists, balms, masks, peels, eye care products, cleansers, and exfoliators. Each row has a product type and its full ingredient list. This grounds the LLM's suggestions in actual commercial formulations rather than hallucinated ingredient combinations.


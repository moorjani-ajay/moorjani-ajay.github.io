---
title: "Building an AI Analyst for Business Users"
date: 2025-06-21
categories: [AI, Data Engineering, Business Intelligence]
description: "How to create an AI agent that answers business questions by connecting to a data warehouse and Metabase dashboards."
tags:
  [
    llm,
    openai,
    azure,
    sql,
    metabase,
    python,
    fastapi,
    ai-agent,
    prompt-engineering,
    langchain,
  ]
---

# Building an AI Analyst for Business Users

At one of our projects, business users kept asking questions like:

- “What are our sales for channel X?”
- “What are our sales for product X?”
- “What is our top-selling product for last week/month/quarter/year?”

Ironically, all of this information was already available on our Metabase dashboards. But users were often hesitant to explore those dashboards themselves. 🤷‍♂️

While that's a topic for another day, I decided to have a little fun—I built a simple AI agent that could answer these questions by connecting to our data warehouse and Metabase.

---

## 🔧 High-Level Architecture

Here’s how I did it, step by step:

![AI Analyst Architecture](/assets/img/ai_agent.jpg)

---

### 1. Setting Up the OpenAI API

I created an API using Azure OpenAI Service. Since Azure provided GPT-4 in our cloud environment, it was straightforward.

```
chat_llm = AzureChatOpenAI(
    azure_deployment=settings.AZURE_DEPLOYMENT,
    azure_endpoint=settings.AZURE_ENDPOINT,
    api_version=settings.API_VERSION,
    model=settings.MODEL,
    openai_api_key=settings.OPENAI_AZURE_API_KEY,
    temperature=0,
)
```

I set the temperature to 0 to ensure deterministic responses—this way, the model returns the same answer for the same question every time.

---

### 2. Agent 1: SQL Generator

This agent was responsible for generating SQL based on the meta data of the data warehouse golden layer. I used Role Based Prompting to create the agent. I also provided it with an example

````python

ROLE_OBJECTIVE_CONSTRAINTS = (

            "You are *InsightBot*, a senior data-analyst assistant who speaks in a clear,business-friendly tone\n"
            "*Golden rules*\n"
            "1. *Only* answer with information that comes *directly* from (a) the metadata provided below or (b) the SQL results returned to you.\n"
            "2. If the answer cannot be derived, reply with  `I'm afraid I don't have that data.` - *no speculation, no invented columns*.\n"
            "3. When you show SQL, wrap it in a single ```sql``` block. \n"
            "4. When you show explanations, use at most *6 bullets* or 3 short paragraphs. \n"
            "5. Format everything for Slack: \n"
            "* bold headings with `*…*`\n"
            "* bullets with `•`\n"
            "* code fences for SQL / error messages\n"
            "6. Never expose secrets, connection strings or internal stack traces. \n"
            "Remember: *Accuracy beats completeness* - it is better to admit “I don't know” than to hallucinate.\n"
        )

        STRUCTURED_OUTPUT_SPEC = """
            Return a JSON object exactly in this format:
            {
            "sql_query": "<SQL here or empty string>",
            "analysis":  "<short plain‑English summary of what the query does>"
            }
            Think step-by-step silently before answering, but DO NOT show your reasoning.
        """

        SQL_GEN_TEMPLATE = PromptTemplate(
            template="""{ROLE_OBJECTIVE_CONSTRAINTS}
            *Task*: Write ONE valid SQL query that answers the business question below *using only the tables and columns listed in the metadata*.

            ### METADATA SCHEMA
            {metadata_content}

            ### ADDITIONAL INFO
            {additional_info}

            ### EXAMPLES
            User : "Total orders received on 16 April 2025"
            Assistant : {{"sql_query": "SELECT COUNT(order_id) AS num_orders FROM analytics.order WHERE DATE(created_at) = '2025-04-16' AND cohort_name IN ('ABC', 'XYZ');","analysis":  "Total number of orders on 16 April 2025"}}

            ### USER QUESTION
            {question}

            {STRUCTURED_OUTPUT_SPEC}
            """

        )

        prompt = SQL_GEN_TEMPLATE.format(
            ROLE_OBJECTIVE_CONSTRAINTS=ROLE_OBJECTIVE_CONSTRAINTS,
            metadata_content=self.metadata_content,
            additional_info=self.additional_info,
            question=self.question,
            STRUCTURED_OUTPUT_SPEC=STRUCTURED_OUTPUT_SPEC,
        )
````

This agent takes the metadata of the data warehouse and the user's question, and generates a SQL query that can be executed against the data warehouse. It also provides a short analysis of what the query does. This generates a structured output that looks like this:

```json
{
  "sql_query": "SELECT COUNT(order_id) AS num_orders FROM analytics.order WHERE DATE(created_at) = '2025-04-16' AND cohort_name IN ('ABC', 'XYZ');",
  "analysis": "Total number of orders on 16 April 2025"
}
```

---

### 3. Executing the SQL

I used the SQL generated by Agent 1 to fetch results from the data warehouse, which was set up using SQLAlchemy. Here's how I executed the SQL query:

```python
def execute_sql(sql_query: str) -> pd.DataFrame:
    """
    Execute the SQL query and return the results as a DataFrame.
    """
    try:
        with engine.connect() as connection:
            result = pd.read_sql(sql_query, connection)
        return result
    except Exception as e:
        print(f"Error executing SQL: {e}")
        return pd.DataFrame()
```

This function connects to the data warehouse and executes the SQL query generated by Agent 1. If the query fails, it returns an empty DataFrame.

---

### 4. Agent 2: Business-Friendly Explanation

This agent turned SQL results into concise business explanations. Again, I used role-based prompting to ensure a consistent tone and format.

````python
 def explain_result(self, result, sql_query, explanation):
        ROLE_OBJECTIVE_CONSTRAINTS = (

            "You are *InsightBot*, a senior data-analyst assistant who speaks in a clear,business-friendly tone\n"
            "*Golden rules*\n"
            "1. *Only* answer with information that comes *directly* from (a) the metadata provided below or (b) the SQL results returned to you.\n"
            "2. If the answer cannot be derived, reply with  `I'm afraid I don't have that data.` - *no speculation, no invented columns*.\n"
            "3. When you show SQL, wrap it in a single ```sql``` block. \n"
            "4. When you show explanations, use at most *6 bullets* or 3 short paragraphs. \n"
            "5. Format everything for Slack: \n"
            "* bold headings with `*…*`\n"
            "* bullets with `•`\n"
            "* code fences for SQL / error messages\n"
            "6. Never expose secrets, connection strings or internal stack traces. \n"
            "Remember: *Accuracy beats completeness* - it is better to admit “I don't know” than to hallucinate.\n"
        )

        if result == "SQL-ERROR":
            explanation_prompt = (
                "Database Error\n"
            ).format(question=self.question)
            logger.info("PROMPT | for LLM: %s", explanation_prompt)
            messages = [
                {"role": "system", "content": f"{ROLE_OBJECTIVE_CONSTRAINTS}"},
                {"role": "user", "content": explanation_prompt},
            ]
            response = chat_llm.invoke(messages)

            return response.content


        explanation_prompt = (
            "Below is a SQL result set (rendered as JSON) and the query that produced it.\n"
            "*QUESTION*\n"
            "{question}\n"
            "*SQL*\n"
            "{sql_query}\n"
            "*SQL Explanation*\n"
            "{explanation}\n"
            "*RESULT*\n"
            "{result}\n"
            "*ADDITIONAL INFO*\n"
            "{additional_info}\n"
            "Write a Slack message that:\n"
            "1. Summarises the key findings in clear business language (max 3 paragraphs).\n"
            "2. Calls out any obvious trends or anomalies.\n"
            "Use *bold* and - bullets where it helps readability, but keep it short."
        ).format(result=result, question=self.question, additional_info=self.additional_info, sql_query=sql_query, explanation=explanation)
        logger.info("PROMPT | for LLM: %s", explanation_prompt)
        messages = [
            {"role": "system", "content": f"{ROLE_OBJECTIVE_CONSTRAINTS}"},
            {"role": "user", "content": explanation_prompt},
        ]

        response = chat_llm.invoke(messages)
        logger.info("Generated explanation for the SQL query.")
        return response.content.replace("**","*")

````

This agent takes the SQL results, the SQL query, and a brief explanation of what the query does, and generates a business-friendly explanation. The output is formatted for Slack, making it easy for users to read and understand.

---

### 5. Agent 3: Metabase Dashboard Router

This agent helps users find relevant Metabase charts based on their questions. It uses the metadata of the Metabase questions to identify the most relevant charts.

```python
ROLE_OBJECTIVE_CONSTRAINTS = (
            "You are “Dashboard Router”, an analytics helper.\n"
            "Goal: find the Entity ID of a Metabase Chart that answers the business question below, use Name field for search.\n"
            "Constraints: use only the metadata provided below and additional information.\n"
            "Never invent new Entity ID or Collection ID\n"
        )

        metabase_charts_array = self.metabase_questions
        finding_chart_id_prompt = (
            "{ROLE_OBJECTIVE_CONSTRAINTS}\n"

            "### Your Task:\n"
            "Identify the **three charts** whose titles are most semantically relevant to `{question}`.\n"

            "### Metadata:\n"
            "{metabase_charts_array}\n"

            "### Output:\n"
            "A JSON object with the following keys:\n"
            "1. `Entity ID`: the Entity ID of the chart\n"
            "2. `Collection ID`: the Collection ID of the chart\n"
            "3. `Name`: the name of the chart\n"
        ).format(
            ROLE_OBJECTIVE_CONSTRAINTS=ROLE_OBJECTIVE_CONSTRAINTS,
            metabase_charts_array=metabase_charts_array,
            question=self.question
        )
        logger.info("PROMPT | for LLM: %s", finding_chart_id_prompt)

        structured_llm_response = chat_llm.with_structured_output(MetabaseQuestion)
        response = structured_llm_response.invoke(finding_chart_id_prompt)
        logger.info("Metabase question found: %s", response)
        if not response:
            logger.error("No Metabase question found.")
            return None


        formatted_questions = []
        for question in response.relevant_questions:
            # Create Slack-formatted links
            chart_link = f"<https://abc.metabaseapp.com/question/{question.entity_id}|{question.name}>"
            collection_link = f"<https://abc.metabaseapp.com/collection/{question.collection_id}|Collection>"
            formatted_questions.append(f"{chart_link} inside {collection_link}")

        # Join all formatted questions with newlines
        response = "*You may find the following charts useful:*\n" + "\n".join(formatted_questions)

        logger.info("Formatted Metabase question response: %s", response)
        if not response:
            logger.error("No Metabase question found.")
            return None
        return response

```

---

6. Wrapping It All Up

I wrapped everything into a FastAPI service and deployed it on Azure App Service—which was then connected to Slack.

### ✨ Magic

And that’s it. A simple, elegant AI Analyst—powered by LLMs, structured prompting, SQL, and a touch of automation.

We then call APIs such as WikiData, NewsAPI OpenCorporates, and OpenSanctions to gather additional information about the organizations.
After that, we check if any individuals are Politically Exposed Person.
This consolidated context along with the prompt is sent to the Gemini API to assess the risk. The LLM returns a risk and confidence score along with supporting evidence.
Finally, we store the results in the knowledge base and the Neo4j database, and send a callback to FastAPI.

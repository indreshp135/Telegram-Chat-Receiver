Here’s your idea explained in 2–3 short paragraphs:


---

The idea is to automate the GCP IAM intake process using a Gen AI-powered chatbot. Right now, this process involves collecting inputs manually, validating them, writing Terraform scripts, and raising change requests. Instead, a chatbot can interact with users, collect all required details, validate them using existing documentation, and prepare everything needed for the intake.

Once the chatbot has the necessary data, it passes the information to a general-purpose Change Request Bot. This bot creates a ServiceNow change request, handles approvals, performs pre-validation, generates the Terraform script, executes it, and follows up with teams to close the request. It also sends alerts in case of errors and manages all communication through email or chat.

The system is designed to be modular. The Change Request Bot is reusable and can be connected to other bots, like an AWS IAM bot or application access bot. This approach standardizes and automates the entire IAM change process, making it faster, more accurate, and easy to scale.


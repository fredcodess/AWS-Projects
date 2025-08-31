#  Set Up a RAG Chatbot in Amazon Bedrock

This project walks through building a **Retrieval-Augmented Generation (RAG) Chatbot** using **Amazon Bedrock**.  
It covers setting up a Knowledge Base, connecting it to your documents in **S3**, embedding with **Titan Text Embeddings v2**, and testing the chatbot with **Meta Llama models**.  

---

## Overview
A **RAG chatbot** enhances a Large Language Model (LLM) with your own data:  
- Documents are stored in **Amazon S3**.  
- Embeddings are generated using **Titan Text Embeddings v2**.  
- Indexed in **Amazon OpenSearch Serverless**.  
- Chat powered by **Llama models in Bedrock**.  

---

## Architecture
![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/rag-archi.png?raw=true)

---

##  Steps  

### 1. Setting Up a Knowledge Base  
- In **Amazon Bedrock Console**, create a new **Knowledge Base (KBase)**.  
- This acts as the library for your chatbot’s data.  

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/rag-kb-create-1.png?raw=true)


---

### 2. Store Your Documents in S3  
- Create an S3 bucket.  
- Upload your documents (PDF, TXT, etc.) into the bucket.  

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/rag-s3-uploaded-files.png?raw=true)

---

### 3. Connect a Data Source to Your Knowledge Base  
- In **Bedrock → Knowledge Base → Data Source**, select your S3 bucket.  
- Leave default **Parsing** and **Chunking strategy**.  
- For embeddings: choose **Titan Text Embeddings v2**.  
- Vector Store: **Quick Create → Amazon OpenSearch Serverless**.  
- Review & Create.  

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/rag-kb-create-2.png?raw=true)
![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/rag-kb-create-3.png?raw=true)


---

### 4. Get Access to AI Models  
Grant access to these models in **Amazon Bedrock → Model Access**:  
- ✅ Titan Text Embeddings v2  
- ✅ Llama 3.1 8B Instruct  
- ✅ Llama 3.3 70B Instruct  

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/rag-3-ai-models.png?raw=true)

💰 **Cost Estimate (as of setup):**  
- Titan Text Embeddings v2 → **£0.00002 / 1,000 tokens**  
- Llama 3.1 8B Instruct → **£0.00022 / 1,000 tokens**  
- Llama 3.3 70B Instruct → **£0.00072 / 1,000 tokens**  

> **Note:** Running this project costs **< £0.01 GDP**.  

---

### 5. Sync Your Knowledge Base  
- Select your Knowledge Base in **Bedrock Console**.  
- On the right panel, select your **AI Model (Llama 3.1 8B Instruct)**.  
- Go to **Data Sources → select your S3 source → Sync**.  

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/rag-s3-sync.png?raw=true)

---

### 6. Test Your Chatbot  
- Inside **Knowledge Base → Test Section**, type your query.  
- Ask one question **about your data** and one **outside your data**.  
- Observe personalized RAG responses vs generic fallback answers.  

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/rag-bedrock-model-for-test.png?raw=true)
![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/rag-bedrock-test-related.png?raw=true)
![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/rag-bedrock-test-not-related.png?raw=true)

---



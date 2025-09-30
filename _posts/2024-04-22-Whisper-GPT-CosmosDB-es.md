---
title: Whisper + GPT. Information Extraction from Recorded Conversations with Azure Functions in Python, and Storing Results in CosmosDB
date:  2024-04-23 10:44:00 -0500
categories: [GenAI, Development, Whisper]   
tags: [azure, development, GPT, openai, python, vscode, cosmosdb, whisper, speech to text, azure functions]
image:
  path: /assets/img/posts/2024-04-22/header.png
---

>Read this article in: [Español](https://warnov.com/@whisper-gpt-post-es), [English](https://warnov.com/@whisper-gpt-post), [Português](https://warnov.com/@whisper-gpt-post-pt)

The ability to quickly and accurately extract information from recorded conversations is a powerful asset for businesses across various industries. Leveraging cutting-edge technologies such as Azure OpenAI’s Whisper and GPT-4 models can transform raw audio into actionable insights. This blog will guide you through a practical scenario of processing recorded conversations using Azure OpenAI services to extract valuable information, which is then stored in CosmosDB, while describing how to adapt this process to a microservices strategy using an example with Azure Blob Storage–triggered functions.

## Scenario Description
Imagine a sales team conducting numerous phone interviews with potential buyers to assess their interest in your products. Each call is full of valuable information—such as the buyer’s name, location, and specific product interests—that are crucial for tailoring future marketing and sales strategies.

### Goal
Extract structured information from each recorded call to better understand potential buyers’ preferences and needs, and then store this information in a database for easy access and analysis by the sales team.

## Implementation Steps Using Azure OpenAI Whisper and GPT-4 Models:

> **Note:** This scenario is implemented in the [GitHub repository](https://warnov.com/@whisper-gpt) that supports this blog post.

Here’s a high-level view of the flow:

![High level view of the flow](/assets/img/posts/2024-04-22/highLevelFlow.png)

1. **Call Recording**:
   - Sales representatives conduct interviews with potential buyers. These calls are recorded with the consent of all participants and stored as audio files in Azure Blob Storage.

2. **Analysis Trigger**:
   - An Azure Blob Storage–triggered function is configured. Each time a new audio file (call recording) is uploaded, this function is automatically triggered.

3. **Audio Processing**:
   - The Azure function first reads the byte stream of the audio recording. It then uses the `NamedBytesIO` class to properly handle the byte stream, ensuring that it includes a filename with the `.wav` extension required by the Whisper API.

4. **Audio Transcription**:
   - The modified byte stream is sent to Azure OpenAI’s Whisper model, which transcribes the audio into text. This transcription converts spoken language into written text, capturing all details mentioned during the call.

5. **Structured Data Extraction**:
   - Once transcribed, the text is passed to the GPT-4 model. Using a predefined prompt, GPT-4 analyzes the transcription to extract structured information such as the customer’s name, geographic location, and product interests.

6. **Data Enrichment**:
   - The function adds metadata to the extracted JSON to create a more useful document.

7. **Storing Results**:
   - The extracted information, together with the transcription, is grouped into an `AnalysisResult` object. This object includes all relevant details and metadata about the call, such as date and time.
   - These structured data are then stored in Azure CosmosDB. Each entry is indexed by the call date and includes identifiers to help the sales team efficiently retrieve and analyze the data.

8. **Additional Processing**:
   - Perform any further processing now that the structured data is available in CosmosDB.

### Architecture
This is the proposed architecture for the solution:

![Whisper + GPT + Azure Functions + CosmosDB: Architecture Integration](/assets/img/posts/2024-04-22/architecture.png)

## Handling File Metadata in Azure Blob–Triggered Functions for Compatibility with OpenAI Whisper Model:

When working with audio data in Python, especially with OpenAI’s Whisper model for transcription, it is crucial that the data not only be in file stream format but also include metadata such as the filename and extension. This requirement is essential because Whisper relies on these metadata—particularly the file extension—to correctly handle the audio data based on its format (e.g., .wav, .mp3). However, when working with Azure Blob–triggered functions, there is a notable complication: the data returned by these triggers typically consist of a raw byte stream that lacks these necessary metadata, including the filename and extension.

To solve this problem and ensure compatibility with the Whisper model, you can use a solution that involves creating a custom wrapper class that mimics a full file stream with the required metadata attributes. For example, the [`NamedBytesIO`](https://github.com/warnov/whisper-gpt/blob/master/common/named_bytes_io.py) class can be defined to extend Python’s [`io.BytesIO`](https://docs.python.org/3/library/io.html#binary-i-o), allowing it to not only carry the byte stream but also simulate having a filename and extension. Here’s how it can be implemented:

```python
import io

class NamedBytesIO(io.BytesIO):
    def __init__(self, buffer, name):
        super().__init__(buffer)
        self.name = name

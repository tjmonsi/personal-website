---
description: "Prompt for a report of intent overlap in the Dialogflow exported to the repository"
applyTo: 'kollab-dialogflow-tool/structures/bdo-ccai-uat-01/intents/**/trainingPhrases/fil.json'
---

# Analyze the Intent's Training Phrases for Overlap

## Objective

The goal of this task is to analyze the training phrases of the intent for potential overlap with other intents. Overlapping training phrases can lead to confusion for the Dialogflow agent, resulting in incorrect intent recognition.

## Directory Structure

Training phrases for each intent are stored in a `fil.json` file under the `trainingPhrases` folder of the intent's directory. The intent's directory is typically named after the intent itself (e.g., `book-flight`) and is located within the `intents` folder of the Dialogflow agent.

## Steps to Analyze Intent Overlap

1. **Identify the Intent**: Start by identifying the intent you are analyzing. This will be the name of the folder (usually formatted as `intent-name` and parent folder of the `trainingPhrases/fil.json` file) associated with the `fil.json` file you are reviewing.
2. **Review Training Phrases**: Open the `fil.json` file and review the training phrases listed under the `trainingPhrases` key. Take note of the phrases and their variations.
3. **Compare with Other Intents**: Compare the training phrases of the current intent with the training phrases of other intents in the same Dialogflow agent. Look for phrases that are similar or identical across different intents.
4. **Identify Overlaps**: If you find training phrases that are identical or very similar (e.g., "book a flight" vs. "book a flight to New York"), note these as potential overlaps. Consider the context in which these phrases are used and whether they could lead to ambiguity in intent recognition.
5. **Disregard Overlaps on the Same Intent**: If you find similar or identical training phrases within the same intent, these should not be considered overlaps, as they are part of the same intent's training data.
6. **Disregard Overlaps on the Same Intent in Different Languages**: If you find similar or identical training phrases across different language versions of the same intent, these should not be considered overlaps, as they are part of the same intent's training data in different languages.
7. **Document Findings**: Document any identified overlaps, including the specific training phrases and the intents they belong to. This documentation will be useful for further analysis and for making recommendations on how to resolve the overlaps.


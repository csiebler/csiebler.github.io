---
title: "5 tips for writing better prompts in Azure OpenAI Studio, OpenAI Playground and ChatGPT"
date: 2023-02-17
---
## Introduction video

This vide post shares 5 tips you can use to write better and more effective prompts in [Azure OpenAI Service](https://azure.microsoft.com/en-us/products/cognitive-services/openai-service/), OpenAI Playground and in ChatGPT:

{{< youtube id="leLPB3UjJlU" >}}

The the sections below for copy/pasting the example prompts.

## Examples

### Tip 1
```
Classify Microsoft, UBS, FedEx, Clemens Inc. into their verticals.
Also add the revenue for 2020. If you don't know certain piece of information, say "N/A". 
```

### Tip 2

JSON:
```
Classify Microsoft, UBS, FedEx, Clemens Inc. into their verticals.
Also add the revenue for 2020. If you don't know certain piece of information, say "N/A".
Answer in JSON using the keys company, vertical, revenue. Make it an array.
```

YAML:
```
Classify Microsoft, UBS, FedEx, Clemens Inc. into their verticals.
If you don't know certain piece of information, say "N/A".
Also add the revenue for 2020.
Answer as YML with keys company, vertical, revenue. Do proper newline formatting.
```

CSV:
```
Classify Microsoft, UBS, FedEx, Clemens Inc. into their verticals.
If you don't know certain piece of information, say "N/A".
Also add the revenue for 2020.
Answer as CSV and print the header "company, vertical, revenue".
```

Markdown as table:
```
Classify Microsoft, UBS, FedEx, Clemens Inc. into their verticals.
If you don't know certain piece of information, say "N/A".
Also add the revenue for 2020.
Answer in markdown using a table.
```

### Tip 3
```
From the following NDA, please extract the following information:

- When the contract was signed
- Who signed the contract
- Address of the person signing the contract
- Fine for breaching the contract

Answer in JSON. Use the keys signing_date, name, address, and fine_amount. Format the signing_date as MM/DD/YYYY.

NDA:
On the date of August 17th, 2019, as an employee of Contoso Restaurant, I, Mateo Gomez, residing in 1234 Hollywood Boulevard Los Angeles CA, with social security number: 123-45-6789, hereby declare to fully support and promote the top priorities delegated to me at Contoso Restaurant, and vow to never disclose any information including but not limited to trade secrets, finances, delivery schedules, and recipes.
If I, Mateo Gomez, accidentally or with intent breach the conditions set forth in this contract, understand fully that I shall receive a written termination to the following email address mateo@contosorestaurant.com as well as a fine of up to $10,000.

JSON:
```

### Tip 4
```
Content:
On the date of August 17th, 2019, as an employee of Contoso Restaurant, I, Mateo Gomez, residing in 1234 Hollywood Boulevard Los Angeles CA, with social security number: 123-45-6789, hereby declare to fully support and promote the top priorities delegated to me at Contoso Restaurant, and vow to never disclose any information including but not limited to trade secrets, finances, delivery schedules, and recipes.
If I, Mateo Gomez, accidentally or with intent breach the conditions set forth in this contract, understand fully that I shall receive a written termination to the following email address mateo@contosorestaurant.com as well as a fine of up to $10,000.

Given the content above, please answer the following question. If you can't find the answer in the content, then write "not found"
Q: Was the signer of the contract married?
A:
```

### Tip 5
```
Extract the following information from the news article below.

1. One sentence summary in German
2. All German States

News article:
Microsoft hat eine starke Präsenz im Bereich der Künstlichen Intelligenz (KI). In Deutschland hat das Unternehmen Büros in Bayern, Baden-Württemberg, Berlin, NRW und anderen Bundeslängern. Das Unternehmen hat eine Reihe von Initiativen gestartet, um KI-Technologien zu entwickeln und zu nutzen. Microsoft hat einige der leistungsstärksten KI-Tools auf dem Markt, einschließlich der Azure Machine Learning-Plattform, die es Entwicklern ermöglicht, Machine-Learning-Modelle zu erstellen und zu trainieren. Microsoft hat auch ein KI-Forschungszentrum eröffnet, das sich auf die Entwicklung von KI-Technologien konzentriert. Microsoft hat auch einige Partnerschaften mit anderen Unternehmen geschlossen, um KI-Technologien zu nutzen. Beispielsweise hat Microsoft eine Partnerschaft mit dem chinesischen Unternehmen Baidu geschlossen, um KI-Technologien für die Entwicklung von Autonomen Fahrzeugen zu nutzen. Microsoft hat auch einige KI-Startups gegründet, um neue KI-Technologien zu entwickeln und zu nutzen. Beispielsweise hat Microsoft ein Unternehmen namens Maluuba gegründet, das sich auf die Entwicklung von KI-Technologien zur Verbesserung der Mensch-Maschine-Interaktion konzentriert. Microsoft hat sich zu einem der führenden Unternehmen im Bereich der Künstlichen Intelligenz entwickelt. Mit seinen verschiedenen Initiativen, Partnerschaften und Investitionen in KI-Technologien hat Microsoft gezeigt, dass es entschlossen ist, KI-Technologien voranzutreiben.
```


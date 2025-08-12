## really fast graphrag  
This fork of graphrag implements a more approach to traditional graphrag, it stores data in GraphDB(Neo4j) along with local vector store. Full semantic search(fuzzy text based search + vector search) is used to filter out nodes relevant to a query, then these nodes are used to build context from path between pairs of entities and radial depth based search from entities. Final context is used for prompting. This approach is more efficient than global search in terms of token usage and better vision of graph than local search. The Scenario that I imagined for this search method was cases where we already have a large well-defined dense graph and want to get into details of the relationships in a more reasonable and affordable manner. 
This method only requires Graph Extraction(better yet, bring you own graph),indexing vector embeddings for entities, and indexing relationships and entities in graphDB(neo4j).  

Insights and good Practices: -
1. Currently the traversal is undirected, which means non relevant relationships may be accepted in during path search or radial depth search. To improve context make sure every relationship which has a valuable inverse is explicitly indexed in graphdb and then switch to directed traversal.
eg. 

Bad context: Blood Test -- Has Intent of --> Diagnosis <-- Has Intent of -- X Ray.    

Good context: High Sugar in blood -- Symptom of --> Diabetes <-- causes --Genetics.

Desired Context: High Sugar in blood -- Symptom of --> Diabetes -- caused by --> Genetics.

In the above examples, Undirected Traversal may include a path from `Blood test` to `Xray` which is undesirable, but Directed Traversal may eliminate path between `High Sugar` and `Genetics` which is also undesirable. Note that original data set may not have `caused by` relationship. In above example inverse of `causes` relationship should be added to graphdb.   

2. Depending on depth parameter Number of Path and relevant entities filtered and subsequent context may be too large, it is better to let the user decide which entity they want to use for traversal for more economic viability. 

Usage: Data - SNOMED-CT
```bash
graphrag query --method rffg --query "Spine pain after exercise" --root ragtest 
```
```
.. ommitted context and logs..
SUCCESS: Local Search Response:
--------- Entities Found --------
ID | Title
2f036b1a-df9f-4aba-8110-ba07a229c5af | EXERCISE INDUCED MYALGIA, EXERCISE INDUCED MYALGIA (FINDING)
eda4cf55-92e1-4d02-9845-6815dab54abb | CHRONIC PAIN AFTER SPINAL SURGERY, CHRONIC PAIN FOLLOWING SPINAL SURGERY, CHRONIC PAIN FOLLOWING SPINAL SURGERY (FINDING)
b5626b31-b0a9-4bb6-a899-c2665ccce255 | PAIN ONSET DURING MODERATE EXERCISE, PAIN ONSET DURING MODERATE EXERCISE (FINDING)
52d3d142-e064-41a9-93e1-9f9c86aa9c46 | POST-EXERCISE POSTURAL HYPOTENSION, POSTURAL HYPOTENSION FOLLOWING EXERCISE, POSTURAL HYPOTENSION FOLLOWING EXERCISE (DISORDER)
2fa4a870-4259-4ec5-8761-a618dd1047b5 | DORSAL SPINE - PAINFUL ON MOVEMENT, PAIN ON MOVEMENT OF THORACIC SPINE, PAIN ON MOVEMENT OF THORACIC SPINE (FINDING), THORACIC SPINE - PAINFUL ON MOVEMENT, THORACIC SPINE - PAINFUL ON MOVEMENT (FINDING)
b8a4c3be-4098-48f9-b4c8-64d66d2ac6c5 | PAIN IN CERVICAL SPINE, PAIN IN CERVICAL SPINE (FINDING)
0d8475b2-cfd4-44c3-ad2a-c85292a8df8a | CERVICAL SPINE - PAINFUL ON MOVEMENT, CERVICAL SPINE - PAINFUL ON MOVEMENT (FINDING), CERVICAL SPINE PAINFUL ON MOVEMENT, CERVICAL SPINE PAINFUL ON MOVEMENT (FINDING), PAIN ON MOVEMENT OF CERVICAL SPINE, PAIN ON MOVEMENT OF CERVICAL SPINE (FINDING)
67d6c9e1-616d-4c27-ac2f-28d69bbcce14 | PAIN IN THORACIC SPINE, PAIN IN THORACIC SPINE (FINDING)
0f0c85e4-4a5e-44e2-ae2b-6b6ab3fc81a9 | LUMBAR SPINE - PAINFUL ON MOVEMENT, LUMBAR SPINE - PAINFUL ON MOVEMENT (FINDING), LUMBAR SPINE PAINFUL ON MOVEMENT, LUMBAR SPINE PAINFUL ON MOVEMENT (FINDING), PAIN ON MOVEMENT OF LUMBAR SPINE, PAIN ON MOVEMENT OF LUMBAR SPINE (FINDING)
0413fed0-5bc6-4c5f-ad48-553ed1e258e9 | THORACIC SPINE ACTIVE, THORACIC SPINE INFLAMED, THORACIC SPINE INFLAMED (FINDING)
e28039fb-8888-4670-855a-b1daf85d061b | EXCERCISE BRONCHIAL PROVOCATION TEST, EXCERCISE BRONCHIAL PROVOCATION TEST (PROCEDURE), EXERCISE BRONCHIAL PROVOCATION TEST, EXERCISE BRONCHIAL PROVOCATION TEST (PROCEDURE)
1bc06ada-8cb1-4bb4-82d9-78a5f3103b1e | AFTER EXERCISE, AFTER EXERCISE (QUALIFIER VALUE), AFTER EXERTION, FOLLOWING EXERCISE, POST EXERCISE
d5d9c43e-4ff2-4d59-b477-8f2e39a30c8b | COUGH AFTER EXERCISE, COUGH ON EXERCISE, COUGH ON EXERCISE (FINDING)
9190a022-0a13-4764-872c-873a340844c8 | BALANCE EXERCISES, BALANCE EXERCISES (REGIME/THERAPY), EXERCISE THERAPY: BALANCE
44ebcf0d-08c5-4b1f-a6e4-b2e8534d5dcc | EXERCISES, THERAPEUTIC EXERCISE, THERAPEUTIC EXERCISE (REGIME/THERAPY), THERAPEUTIC EXERCISE (REGIME/THERAPY)(PROCEDURE), THERAPEUTIC EXERCISE, NOS
```

```
graphrag query --method rffg --query "Spine pain after exercise" --root ragtest --entities 44ebcf0d-08c5-4b1f-a6e4-b2e8534d5dcc --entities 52d3d142-e064-41a9-93e1-9f9c86aa9c46
```

```
.. ommitted context and logs..
Therapeutic exercise, classified as a physiotherapy or physical medicine procedure, is defined as a structured physical activity regimen aimed at achieving therapeutic benefits. Its procedural intent is explicitly therapeutic, meaning it is designed to improve health outcomes rather than serve diagnostic or purely preventive purposes \[Data: Entities (356399, 128312); Relationships (1380030, 3303502, 738386, +more)].

A documented clinical association exists between therapeutic exercise and the occurrence of post-exercise postural hypotension — a condition characterized by a sudden drop in blood pressure upon standing after exercise. This link is understood through the shared physiological context: therapeutic exercise affects the cardiovascular system, which is also the anatomical site implicated in post-exercise postural hypotension \[Data: Entities (5527583, 119790); Relationships (5527585)]. In the provided pathway mappings, both conditions are connected via intermediate procedural relationships involving cardiovascular structures and therapeutic intent, indicating that the outcome is a recognized potential post-exercise complication \[Data: Paths Found; Entities (356399, 5527583, 119790); Relationships (1380030, 5527585)].

The cardiovascular system serves as the central anatomical link between the procedure and the disorder. This includes structures such as arteries, veins, and the heart, which are involved both in exercise-induced physiological changes and in the mechanisms that lead to postural hypotension after exertion \[Data: Entities (119790); Relationships (5527585)]. The intermediate links in the data — such as surgical ligation of aneurysms in cardiovascular structures — reinforce the systemic association between therapeutic interventions targeting the cardiovascular system and post-exercise hypotensive outcomes, even though surgical and exercise contexts differ in mechanism \[Data: Entities (64358, 16528); Relationships (6552934, 6512611, +more)].

In clinical terms, this association means that while therapeutic exercise is generally beneficial, practitioners should be aware of the risk of post-exercise postural hypotension, especially in patients with preexisting cardiovascular conditions or those undergoing cardiovascular interventions. Monitoring and gradual cooldown protocols are often recommended to mitigate this risk.

```
Work flow Diagram:- 
<img width="8213" height="4667" alt="image" src="https://github.com/user-attachments/assets/1d9416c6-fd25-4ff9-a7db-396032dd6fff" />

# GraphRAG

👉 [Microsoft Research Blog Post](https://www.microsoft.com/en-us/research/blog/graphrag-unlocking-llm-discovery-on-narrative-private-data/)<br/>
👉 [Read the docs](https://microsoft.github.io/graphrag)<br/>
👉 [GraphRAG Arxiv](https://arxiv.org/pdf/2404.16130)

<div align="left">
  <a href="https://pypi.org/project/graphrag/">
    <img alt="PyPI - Version" src="https://img.shields.io/pypi/v/graphrag">
  </a>
  <a href="https://pypi.org/project/graphrag/">
    <img alt="PyPI - Downloads" src="https://img.shields.io/pypi/dm/graphrag">
  </a>
  <a href="https://github.com/microsoft/graphrag/issues">
    <img alt="GitHub Issues" src="https://img.shields.io/github/issues/microsoft/graphrag">
  </a>
  <a href="https://github.com/microsoft/graphrag/discussions">
    <img alt="GitHub Discussions" src="https://img.shields.io/github/discussions/microsoft/graphrag">
  </a>
</div>

## Overview

The GraphRAG project is a data pipeline and transformation suite that is designed to extract meaningful, structured data from unstructured text using the power of LLMs.

To learn more about GraphRAG and how it can be used to enhance your LLM's ability to reason about your private data, please visit the <a href="https://www.microsoft.com/en-us/research/blog/graphrag-unlocking-llm-discovery-on-narrative-private-data/" target="_blank">Microsoft Research Blog Post.</a>

## Quickstart

To get started with the GraphRAG system we recommend trying the [command line quickstart](https://microsoft.github.io/graphrag/get_started/).

## Repository Guidance

This repository presents a methodology for using knowledge graph memory structures to enhance LLM outputs. Please note that the provided code serves as a demonstration and is not an officially supported Microsoft offering.

⚠️ *Warning: GraphRAG indexing can be an expensive operation, please read all of the documentation to understand the process and costs involved, and start small.*

## Diving Deeper

- To learn about our contribution guidelines, see [CONTRIBUTING.md](./CONTRIBUTING.md)
- To start developing _GraphRAG_, see [DEVELOPING.md](./DEVELOPING.md)
- Join the conversation and provide feedback in the [GitHub Discussions tab!](https://github.com/microsoft/graphrag/discussions)

## Prompt Tuning

Using _GraphRAG_ with your data out of the box may not yield the best possible results.
We strongly recommend to fine-tune your prompts following the [Prompt Tuning Guide](https://microsoft.github.io/graphrag/prompt_tuning/overview/) in our documentation.

## Versioning

Please see the [breaking changes](./breaking-changes.md) document for notes on our approach to versioning the project.

*Always run `graphrag init --root [path] --force` between minor version bumps to ensure you have the latest config format. Run the provided migration notebook between major version bumps if you want to avoid re-indexing prior datasets. Note that this will overwrite your configuration and prompts, so backup if necessary.*

## Responsible AI FAQ

See [RAI_TRANSPARENCY.md](./RAI_TRANSPARENCY.md)

- [What is GraphRAG?](./RAI_TRANSPARENCY.md#what-is-graphrag)
- [What can GraphRAG do?](./RAI_TRANSPARENCY.md#what-can-graphrag-do)
- [What are GraphRAG’s intended use(s)?](./RAI_TRANSPARENCY.md#what-are-graphrags-intended-uses)
- [How was GraphRAG evaluated? What metrics are used to measure performance?](./RAI_TRANSPARENCY.md#how-was-graphrag-evaluated-what-metrics-are-used-to-measure-performance)
- [What are the limitations of GraphRAG? How can users minimize the impact of GraphRAG’s limitations when using the system?](./RAI_TRANSPARENCY.md#what-are-the-limitations-of-graphrag-how-can-users-minimize-the-impact-of-graphrags-limitations-when-using-the-system)
- [What operational factors and settings allow for effective and responsible use of GraphRAG?](./RAI_TRANSPARENCY.md#what-operational-factors-and-settings-allow-for-effective-and-responsible-use-of-graphrag)

## Trademarks

This project may contain trademarks or logos for projects, products, or services. Authorized use of Microsoft
trademarks or logos is subject to and must follow
[Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general).
Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.
Any use of third-party trademarks or logos are subject to those third-party's policies.

## Privacy

[Microsoft Privacy Statement](https://privacy.microsoft.com/en-us/privacystatement)

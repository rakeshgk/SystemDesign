# Understanding the Problem

## Functional Requirements

1. The first thing you'll want to do when starting a system design interview is to get a clear understanding of the requirements of the system. 
2. Functional requirements are the features that the system must have to satisfy the needs of the user.
3. If you are not provided with the functional requirements you will have to determine them yourself. If you're familiar with the product, this task should be relatively straightforward. However, if you're not, it's advisable to ask your interviewer some clarifying questions to gain a better understanding of the system.
4. The most important thing is that you zero in on the top 3-4 features of the system and don't get distracted by the bells and whistles.

## Non-Functional Requirements

1. Non-functional requirements refer to specifications about how a system operates, rather than what tasks it performs.
2. These requirements are critical as they define system attributes like scalability, latency, security, and availability, and are often framed as specific benchmarks such as the system ability to handle 100 Million DAU or respond to queries within 200 ms. 

# The Set Up

## Identify and define Core Entities

1. We start by identifying the core entities. At this stage, it is not necessary to know every specific column or detail.
2. Identifying the core entities will help us come up with APIs that our system should support.

## The API

1. Go through the core functional requirements and define the APIs that are necessary to satisfy them.
2. Usually, the APIs map 1:1 to the functional requirements, but there are times when multiple endpoints are needed to satisfy an individual functional requirement.

# High Level Design

1. Start the design by going one-by-one through the functional requirements and designing a single system to satisfy them.
2. Once this is in place, the design can always be extended during the Deep Dives


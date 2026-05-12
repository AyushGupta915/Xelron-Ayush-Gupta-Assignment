# Part 4: Technical Communication

## Task 4.1: Scenario Response

---

**Reviewer's question:** "Why did you choose this specific PR over the others? What made it comprehensible to you, and what challenges do you anticipate in implementing it?"

---

PR #3877 (web readonly), which came from the beets repository, appealed to me due to several considerations concerning understandability and breadth of coverage.

The PR concerns a problem that is readily understandable – an HTTP API had been allowing destructive actions with no access control at all until somebody spotted this loophole and addressed it. I am not expected to be deeply versed in the inner workings of the beets library model, nor do I have to know about the consumer group assignment algorithm used by Kafka, to comprehend what "default read-only mode" means. The scope of the work encompasses HTTP-related and configuration management issues – two domains with well-understood and stable concepts. In contrast, other PRs such as #3568 (albuminfo/trackinfo classes) demanded knowing details of beets' internal data model and metadata flow through the autotag pipeline, and contributions to the aiokafka repository were related to asyncio socket multiplexing and Kafka record protocols.

Having my background as a web developer and designing RESTful APIs, I feel comfortable working with PR #3877. I am aware of how `app.config` behaves independently of application-level config objects. I can easily relate to the two-config namespace problem mentioned in the PR description. Beets uses its `confuse` YAML config for configuring itself while Flask framework maintains a different namespace for config settings. Such inter-operating details of Flask and beets can be found in Django or any other framework that beets interacts with.

The main difficulties I see with this project are in the tests layer. Beets plugins have a rather complex way of creating a test environment, including generating a full config environment for beets and initializing a Flask test client. These two pieces of code may interact badly, causing tests to fail without a proper setup. Thus, I will spend extra effort studying existing web plugins tests to ensure that new tests are correct and do not pass on account of a wrong config environment being used. Another difficulty lies in completeness. Not only delete but all routes capable of writing should be protected. To overcome this issue, I will examine every single route decorator inside `web.py` file and compare them to the list of routes in documentation.

To overcome the above problems, firstly, I would check the test cases present for the web plugin to see how injection of configuration takes place and the initialization of the Flask test client. Secondly, I would copy the same approach while writing the test cases for the routes. To cover all routes, I would check all routes present in `web.py` to see whether they have any write actions.

On the whole, this PR was selected for the reason that there is a very concrete problem, as well as a straightforward and practical solution to it. The task is related to my previous experience of working with backends and APIs, which made it relevant and interesting.

With my prior experience with REST APIs and configuration handling, I feel that implementing this PR will not be difficult and will allow me to maintain high standards in terms of code quality and testing.
---

*I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words.*

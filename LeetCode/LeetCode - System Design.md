
There is a question bank that stores thousands of programming questions. For each question, users can write, compile and submit code against test cases and get results. User submitted code needs to be persisted. Every week, there is a contest where people compete to solve questions as quickly as possible. Users will be ranked based on their accuracy and speed. We’ll need a leaderboard that shows rankings in real time.



## Functional Requirements

rather than just listing all possible functionalities, your goal is to prioritize based on business impact and technical feasibility




## Core Requirements

1. **View Problems**: Users should be able to view problem descriptions, examples, constraints, and browse a list of problems.
    
2. **Submit Solution**: Users should be able to solve coding questions by submitting their solution code and running it against built-in test cases to get results.
    
3. **Coding Contest**: User can participate in coding contest. The contest is a timed event with a fixed duration of 2 hours consisting of four questions. The score is calculated based on the number of questions solved and the time taken to solve them. The results will be displayed in real time. The leaderboard will show the top 50 users with their usernames and scores.

### Out of Scope

- Authentication
- Authorization
- User management
- Contest history


### Scale Requirements

- Supporting 10k users participating contests concurrently
- There is at most one contest a day
- Each contest lasts 2 hours
- Each user submits solutions 20 times on average
- Each submission executes 20 test cases on average
- User submitted code have a retention policy of 1 month after which they will be deleted.
- Assuming that the storage space required for the solution of each coding question is 10KB.
- Assuming that the read:write ratio is 2:1.


## Non-Functional Requirements


- High availability: the website should be accessible 24/7
- High scalability: the website should be able to handle 10k users submitting code concurrently
- Low latency: during a contest, the leaderboard should be updated in real time
- Security: user code cannot break server


## API Endpoints


#### GET /problems?start={start}&end={end}

Get problems in a range. If start and end are not provided, get all problems with default page size.

```
{ problems: [ { id: string, title: string, difficulty: string } ] }
```

#### GET /problems/{problem_id}


![[Pasted image 20250509154307.png]]


#### POST /problems/{problem_id}/submission

Submit code for a problem and get results.

![[Pasted image 20250509154323.png]]


#### GET /contests/{contest_id}/leaderboard

![[Pasted image 20250509154334.png]]



## 1. View Problems

Users should be able to view problem: descriptions, examples, constraints, and browse a list of problems

1. Client: Frontend application sending HTTP GET requests
2. Problems Service: Backend service with endpoints:


![[Pasted image 20250509154626.png]]


## 2. Submit Solution

- users should be able to solve coding questions by submitting their solution code and running it against built-in test cases to get results

to design the code evaluation we need to consider the high concurrency requirements of 10k users participating in contests simultaneously 

We use a **Code Evaluation Service** as the entry point for code submissions. This service will be responsible for receiving code from users and initiating the evaluation process. However, to handle the high volume of submissions efficiently and securely, we introduce a **Message Queue** as a buffer between the submission process and the actual code execution.


![[Pasted image 20250509154858.png]]

When a user submits a solution, the Code Evaluation Service pushes a message containing the submission details (user ID, problem ID, and code) to the Message Queue. This **decouples the submission process from the execution process**, allowing us to handle **traffic spikes** more effectively.


On the other side of the Message Queue, we have multiple **Code Execution Workers**. These workers continuously poll the queue for new submissions. When they receive a message, they fetch the necessary test cases from the Problems database, **execute the code in a sandboxed environment**, and record the results. Let's compare different execution options for the sandbox environment:



- API Server
- Virtual Machine (VM)
- Container (Docker)
- Serverless Function


We will choose **Container (Docker)** over serverless functions for the faster startup time and less vendor lock-in.

To ensure **security and isolation**, each submission runs in a **separate container**, preventing malicious code from affecting our system or other users' submissions. We can use technologies like **Docker** for this purpose.

After execution, the results (including whether the code passed all test cases, execution time, and memory usage) are saved in the Submissions table.

This design allows us to **scale horizontally** by adding more Code Execution Workers as needed to handle increased load during peak times or popular contests. The use of a Message Queue also provides a level of **fault tolerance** - if a Code Execution Worker fails, the message remains in the queue and can be picked up by another worker.



## 3. Coding Contest

User can participate in coding contest. The contest is a timed event with a fixed duration of 2 hours consisting of four questions. The score is calculated based on the number of questions solved and the time taken to solve them. The results will be displayed in real time. The leaderboard will show the top 50 users with their usernames and scores.

A contest itself is basically four problems with a time window. Running the contest itself is straightforward. we can use a centralized service to manage the contest, Contest Service

![[Pasted image 20250509155438.png]]



## How does the code evaluation service decides if a submission is correct


![[Pasted image 20250509155823.png]]


- **Resource Limitation**: Limit the resources that each code execution can use, such as CPU, memory, and disk I/O.
- **Time Limitation**: Limit the execution time of each code execution.
- **System Call Limitation**: Use technologies like `seccomp` to limit the system calls that the code can make.
- **User Isolation**: Run each user's code in a separate container.
- **Network Isolation**: Limit the network access of each code execution.
- **File System Isolation**: Limit the file system access of each code execution.



### How to implement a leaderboard that supports top N queries in real time?



### Redis Sorted Set

Uses a sorted set data structure to maintain scores. Each user is a member of the set, with their score as the sorting criteria.

- Extremely fast read and write operations
- Built-in ranking and range queries
- Scalable and can handle high concurrency

#### Sorted Set Internals

Internally, the sorted set is implemented as a hash map and a [skip list](https://en.wikipedia.org/wiki/Skip_list). The hash map stores the mapping between the user IDs and the scores. The skip list keeps the scores in ascending order. Skip list is not a common data structure, so it's worth explaining it here.

- **Skip list**: A skip list is a **probabilistic data structure** that allows for fast search, insert, and delete operations. It's a series of linked lists with additional pointers to intermediate nodes, allowing for efficient traversal and search operations. **Functionally, it's similar to a balanced binary search tree, but it's implemented using linked lists and randomization**. The time complexity of search, insert, and delete operations is **O(log n)** on average. The randomization is used to ensure that the skip list is balanced, so the performance is good even if the list is not balanced. This is similar to how binary search trees need to be balanced by rotating nodes. Finally, the skip list can return top N in **O(N(log M)) time** where N is the number of elements to return and M is the number of elements in the list.

Here's an example of a skip list:


```
Level 3:  [ 1 ]-------------------------------------> [ 20 ]
            |                                           |
Level 2:  [ 1 ]-----------------> [ 7 ]-------------> [ 20 ]
            |                       |                   |
Level 1:  [ 1 ]-----> [ 3 ]-----> [ 7 ]------> [ 15 ]-[ 20 ]
            |           |           |             |     |
Level 0:  [ 1 ] [ 2 ] [ 3 ] [ 5 ] [ 7 ] [ 10 ] [ 15 ] [ 20 ] [ 25 ]
```

- **Levels**: Higher levels have fewer elements, allowing "skips" to speed up searching. For example, level 3 can directly skip from node 1 to 20, avoiding unnecessary steps.
- **Nodes**: Each node holds a value (e.g., 1, 3, 7, etc.). At level 0, every node is linked like a standard linked list.
- **Links**: Vertical links connect nodes across different levels, while horizontal links connect nodes at the same level.

The search works by starting at the highest level and moving down as needed:

- Start at the highest level (e.g., Level 3).
- Move right until the next node’s value is greater than or equal to the target.
- Move down a level if the next value is too large.
- Repeat this process at each level, moving right as far as possible, then down, until the target is found or confirmed absent.

Example: Searching for 15

- Start at node 1 on Level 3, drop down since 20 > 15.
- On Level 2, move from 1 to 7, then drop down again.
- On Level 1, move to 15 and stop.

This approach gives an average O(log n) time complexity by efficiently skipping unnecessary nodes.

The skip list is a series of linked lists with additional pointers to intermediate nodes, allowing for efficient traversal and search operations.

To get the leaderboard, i.e. get the top `N` users, we can go down to the level 0 and find the last element in the list, and then perform a backward iteration to get the top `N` users (the linked lists in a skip list are doubly linked).

In Redis, this is implemented as `ZSET`. We use `ZADD` to add a user and its score, and use `ZRANGE` to get the top `N` users.

#### Leaderboard Example

Here's a simple example:

`ZADD leaderboard 100 user1 ZADD leaderboard 200 user2 ZADD leaderboard 300 user3 ZREVRANGE leaderboard 0 1 # returns the top 2 users`

### How to handle a large number of contestants submitting solutions at the same time?

Handling a large number of contestants submitting solutions simultaneously presents several challenges, particularly during peak times like the beginning and end of a contest. We can address this using a [two-phase processing](https://systemdesignschool.io/fundamentals/two-stage-processing) approach combined with other strategies:

1. **Message Queue**: Use a message queue to decouple the submission process from the execution process. This allows us to buffer submissions during peak times and process them as resources become available.
    
2. **Pre-scaling**: Since auto-scaling may not be fast enough to handle sudden spikes, pre-scale the execution servers before the contest starts. This is especially important for the first and last 10 minutes of a contest when submission rates are typically highest.
    
3. **Two-Phase Processing**:
    
    - **Phase 1 (During Contest)**: Implement partial evaluation
        - Run submissions against only about 10% of the test cases.
        - Provide a partial standing for users, giving them immediate feedback.
    - **Phase 2 (Post-Contest)**: Complete full evaluation
        - After the contest deadline, run the submissions on the remaining test cases.
        - Determine the official results based on full evaluation.

This two-phase approach:

- Reduces the load on the system during the contest.
- Allows for more accurate final results.
- Is similar to the approach used by platforms like Codeforces for large contests.

4. **Efficient Resource Allocation**: Optimize resource allocation by scaling down during less active periods of the contest and scaling up for the final rush.
    
5. **Prioritization**: Implement a prioritization system in the message queue to ensure that at least one submission from each user is processed quickly, providing everyone with some feedback.
    
6. **User Communication**: Clearly communicate to users that their submissions are being partially evaluated during the contest and that final results will be available after full evaluation post-contest.
    

This approach, centered around the [two-phase processing technique](https://systemdesignschool.io/fundamentals/two-stage-processing), balances immediate feedback for users with efficient resource utilization. While it means that final results won't be available immediately after the contest ends, it allows for handling a much larger number of concurrent submissions with existing resources. The trade-off between real-time complete results and system scalability is often necessary for large-scale coding competitions.




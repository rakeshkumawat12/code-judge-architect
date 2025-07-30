Code Judge - service that "judges" your code submission. When you submit a solution for a DSA problem in any platform like Scaler, Leetcode, Codechef, GFG and many more. They have to take our code and run it through hundreds of testcases to evaluate whether it is correct or not.

We will be understanding through Scaler example.

## Situation
### Q: What does the appserver need to evaluate a code submission?
1. User info (auth, already solved this problem or not, part of course or not, ..) 
2. Problem info (topic, total score, memory limit, time limit, number of testcases, ..) 
3. Testcase data (input file, expected output file)

### Q: How large can this data be (in the avg/worst case)?
User info / Problem info - a few KBs **(SQL database - Amazon RDS)**  
Testcase data - 1 to 2 GBs **(File storage - Amazon S3)**

For a problem we run your code through 100 testcases.   
Imagine that the problem is about sorting the array.  
Each testcase comprises of a large array (N = 10^6).  
Each expected output for each testcases is also a large array (N = 10^6)

```
100 testcases * (10^6 integers / testcase) * (8 bytes / integer)
= 100 * 10^6 * 8 bytes
= 800 MB
~ 1 GB
```
This is for the input file. similar size for the output file.


### Q: What is the total size of testcase data in S3?    
Scaler has around 3, 000 problems   
so around **3000 problem * (1GB testcase data / problem)**  
**= 3TB** of testcase data

Is this data too large?  
Not really - storage is cheap.  
3TB is too large to fit completely in RAM. But can easily fit on HDD.

### **Q: How many problems are being solved on any given day?**  
Around 100 200 different problems are being solved by the Scaler students on any given day. We're running multiple batches in parallel. Each batch might be solving the problems of the current (or last) class. The batches also overlap in timelines.

![IMAGE1](./Assets/img1.png)

### **Q: Should we store these testcases in a CDN?**  
No! Backend services will never access data from the CDN  
1. CDNs are client facing. 
2. CDNS reduce latency for the client, by serving the data from an (edge) server close to the user. 

### Need for caching
**Q: Do we want to transfer 1GB of data over the network for each request?**  
Absolutely No.  
Therefore, we need a cache!  

Local vs Global
• Global Cache: doesn't really solve our problem
- yes, it will be available to all app servers (so our app servers can be stateless)
- yes, the data is large, so global makes more sense?
- NO - it doesn't solve our problem at all - we still have to transfer 1GB of testcase data for each request - from the cache server to the app server
• earlier we were transferring from S3 to app server - expensive!
• now we're transferring from cache to app server - expensive!

![IMAGE2](./Assets/img2.png)


Local Cache: each server caches some testcases in the local HDD
- the request will come to the app server
- the app server will hit the SQL db to fetch the user/problem info (small data)
- if the problem's testcases are already cached in the appserver's HDD, it will just use those testcase files (no n/w overhead)
- if not cached, it will download the files from S3, and cache it locally (on its HDD), and then use it
- any subsequent requests for the same problem will now use the testcases from the HDD 

![IMAGE3](./Assets/img3.png)


POST /evaluate-submission

```
fn check_solution (request):
    pid = request.solution.problem_id
    uid = request.auth_token.uid
    user_info, problem_info = fetch_from_SQL (user_id, pid)

    if not file_exists(`{pid}_in.txt`):    // caching logic 
        download_from_s3([`{pid}_in.txt`, `{pid}_out.txt`]) 
        // this function will save it on the local HDD
        
    inputs = read_file(`{pid}_in.txt') // these reads are from the local HDD
    expected_outputs = read_file(`{pid}_out.txt`)

    return evaluate (solution.code, inputs, expected_outputs)
```

### Single vs Distributed    
N/A - single vs distributed only applies for global caches
Local cache is distributed automatically, because there are multiple app server


### Invalidation Algorithm
**When is invalidation applicable?**  
if the data never changes, do we need to invalidate?
Invalidation is only applicable when the data changes.

**Q: Can the testcases change?**  
New problems can be added but that is not a data updation.  

Can the testcase data for an existing problem be updated?  
Yes.  
Because our problem setters are not Gods. they can make mistakes.  
It is possible that
• our editorial solution was incorrect
- we've to fix the solution and the expected_output.txt file
• our solution was correct, but our testcases were not "tight enough"
- our problem required a O(n log n) solution, but the testcases let the  O(n2) solutions pass as well.  
- so we've to fix both the input.txt and the
expected_output.txt file

**Q: How frequently will the testcases change?**   
Extremely rarely. We have good problem setters, and a lot of quality assurance to ensure that bad problems are not created. But it happens from time to time.

Let's say for a given problem we might want to update the testcases once a year on average.  
1 update / problem / year on average

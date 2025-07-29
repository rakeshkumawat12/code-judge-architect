Code Judge - service that "judges" your code submission. When you submit a solution for a DSA problem in any platform like Scaler, Leetcode, Codechef, GFG and many more. They have to take our code and run it through hundreds of testcases to evaluate whether it is correct or not.

We will be understanding through Scaler example.

##Situation
###Q: What does the appserver need to evaluate a code submission?
1. User info (auth, already solved this problem or not, part of course or not, ..) 
2. Problem info (topic, total score, memory limit, time limit, number of testcases, ..) 
3. Testcase data (input file, expected output file)

###Q: How large can this data be (in the avg/worst case)?
User info / Problem info - a few KBs **(SQL database - Amazon RDS)**. 
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


# Movie Recommendation System (with Spark)


This project aims to complete the given code and leverage Spark to build a movie recommender system in Scala. 

The Alternating Least Squares (ALS) algorithm in `spark.mllib` was used as the process for prediction. 

## Contents
- [Model-based Collaborative Filtering](#mbf)
- [Pre-requisites](#pr)
- [Cluster testing](#clt)
- [Completion of the code](#cc)
- [Running on local cluster](#rlc)
- [Video Demonstation (Interactive mode)](#vd)
- [Preparation of distributed cluster](#pdc)
- [Running on distributed cluster](#rdc)
- [Results](#res)
- [Discussion on model (Ratings of Scala)](#dm)
- [Self-Check Questions](#sq)
- [Additional Material](#am)

## Model-based Collaborative Filtering (CF) Algorithm<a id="mbf"> </a>
It integrates the user and their choices to build a list of suggestions. These suggestions that are given by this filtering technique are built on the cooperation of several users and their close interests. All the users’ choices are compared, and they receive a recommendation. 

Based on the feedback from different types of rating systems, companies can gather a large amount of information about this feedback and make predictions about your preferences and offer recommendations based on ratings from users that are similar to you.

***FIGURE BELOW*** shows a basic diagram of the collaborative filtering method. 
<a href="https://drive.google.com/uc?export=view&id=<FILEID>"><img src="https://drive.google.com/uc?export=view&id=10rWveVgXKfovGvvImMSWz40NjQyugPPl" style="width: 650px; max-width: 80%; height: auto" title="Click to enlarge picture"></a>

To know more - [Click Here](#am)


## Github
Github repository: https://github.com/ParthKalkar/movie-recommendation-spark


## Prerequisites <a id="pr"> </a>

- We first examined the spark and scala version of the machines in the cluster with the command 
```>>spark-submit --version```
```>>version 3.2.0                  Using Scala version2.12.15```
- We needed to change version in build.sbt file to specified. Moreover, to avoid compilation error *"insecure HTTP request is unsupported"*, we need to change all urls in build.sbt ```http -> https```
- Then we launched local Jupyter notebook with spylon-kernel (for faster and more user friendly compilation)
- Then we created a file ```user_rating.tsv```in ```movielens-modified``` directory, this file contains user input in form ```movie_id rating``` (used when model does not ask for user input)

## Cluster testing <a id="clt"> </a>

- Firstly, one needs to compile the ```.jar``` file with ```package sbt``` command on ```spark-recommendation``` directory
- After prerequisites step, the compilation was successful
<a href="https://drive.google.com/uc?export=view&id=<FILEID>"><img src="https://drive.google.com/uc?export=view&id=1gtxx9i0jhzPFsFVAYDAbRiZAvdyP8tpe" style="width: 650px; max-width: 80%; height: auto" title="Click to enlarge picture"></a>
- Set up a local cluster with command ```vagrant up```
- Start preconfigured hdfs and yarn with ```start-all.sh```
- Check if everything is working with ```hdfs dfsadmin -report && yarn node -list``` command
<a href="https://drive.google.com/uc?export=view&id=<FILEID>"><img src="https://drive.google.com/uc?export=view&id=1hI7LRrlwDEycMiR-WSEclUY7DxwCIW4I" style="width: 650px; max-width: 80%; height: auto" title="Click to enlarge picture"></a>
- If you see all nodes in the output, then proceed with
```
  export YARN_CONF_DIR=hadoop/etc/hadoop
  export HADOOP_USER_NAME=vagrant
```
- Put the files to hdfs
``` 
hdfs dfs -put <local path to movielens-modified> <path on hdfs>
```
- Create a ```user_ratings.tsv``` file to store user inputs
```
touch user_ratings.tsv
hdfs dfs -put user_ratings.tsv
```
- Finally, execute testing command
```
park-submit --master yarn --conf spark.network.timeout=800 --class MovieLensALS <local path to jar> <path on hdfs> -user false
```



## Completion of the code <a id="cc"> </a>
The code given was incomplete to run in interactive manner with 

- **RMSE:**
  Analogously to `rmse(test: RDD[Rating], prediction: RDD[Rating])`, we define `rmse(test: RDD[Rating], prediction: scala.collection.Map[Int, Double])` as follows:
  ```scala=
  def rmse1(test: RDD[Rating], prediction: scala.collection.Map[Int, Double]) = {
    math.sqrt(test
      .map(x => ((x.rating - prediction(x.product)) * (x.rating - prediction(x.product))))
      .reduce(_ + _) / test.count()) 
  }
  ```
  However, due to having ```prediction: scala.collection.Map[Int, Double]``` we need to access the values of columns in a different manner like ```prediction(x.product)```
Prediction in this case is a baseline collection of pairs ```(film, average rating)```.
Thus, this rmse function tries to check the error if we had a 'not-smart' model, that just predicts an average value. 


- **Parse Title:**
  This function is similar to `parseId`:
  ```scala=
    def parseTitle(filmEntry: String) = {
    ",.*,".r findFirstIn filmEntry match {
      case Some(s) => s.slice(1, s.length - 1)
      case None => throw new Exception(s"Cannot parse Title in {$filmEntry}")
    }
  }
  ``` 
  We changed the regular expression that captures everithing beween 2 commas unless these commans are not inside the quotation marks. After we cropp 2 commas from matchin string and save it as a movie title. A screenshot for cases with/without quotations are provided below:
<a href="https://drive.google.com/uc?export=view&id=<FILEID>"><img src="https://drive.google.com/uc?export=view&id=17T2QotPSHzoo9h_d01xUhHauUlKXbp3o" style="width: 650px; max-width: 100%; height: auto" title="Click to enlarge picture"></a>
  

- **Post-processing of recommendations:**
  To filter out movies that the user has seen, we add the following code to the block `if (doGrading)` in `main`:
  ```scala=
    var seen = myRating.filter(x => x.rating>0).map(x => (x.product, x.rating)).countByKey()
    model.predict(sc.parallelize(baseline.keys.filter(x => !seen.keySet.exists(_ == x)).map(x => (0, x)).toSeq))
  ```
  - Here, ```MyRating``` is a user defined RDD of rated movies in form  ```(user_id, movie_id, rating)``` 
  - With ```filter(x => x.rating>0)``` we filter all movies a user has actually seen (i.e gave rating !=0 ).
  - After it, we  map each tuple ```(user_id, movie_id, rating)``` to ```(movie_id, rating)```
  - Then with ```.countByKey()``` we transform it into a set of tuples ```(movie_id, Count)```
  - Now we have a set of seen movies
  - With ```baseline.keys.filter(x => !seen.keySet.exists(_ == x)``` we filter all movies, that are contained in baseline, but not contained in ```seen```
  - Map to movies (0, x), where 0 is a user id, and x is movie id 
  
- **Loading Movie Preferences from the file:**
To load movie preferences from a file we need to change a Grader class and introduce a new argument ```interactive```

   - If ```interactive == true``` the programme will ask user to rate movies in interactive manner, in the opposite case, program will read from a file.
   - We need to change the constructor as follows (now it takes ```interactive``` as an argument)
   ```scala=
   class Grader(path: String, sc: SparkContext, interactive: Boolean)
    ``` 
   - In ```MovieLensALS``` class we need to capture user input for new parameter and create a corresponding grader class 
```scala=
if (args.length != 4) {
      proceed = false
    }
    if (args(1) == "-user")
      try{
        doGrading = args(2).toBoolean
        interactive = args(3).toBoolean #specify one more boolean after -user option
      } catch {
        case e: Exception => proceed = false
      }

    if (!proceed) {
      println(usageString)
      sys.exit(1)
    }

    // return tuple
    (ratingsPath, doGrading, interactive)
  }
  ```
   - Changes in ```toRDD``` function:
  ```scala=
    def toRDD = {
      if (interactive == true) {
        sc.parallelize(this.graded.map{x => Rating(0, x._1, x._2)})
      }
      else {
      val films = sc
      .textFile(path + "/user_rating.tsv")
      .map{_.split("\t")}
      .map{x => (x(0).toInt, x(1).toDouble)}
      .collect()
      .toSeq
        sc.parallelize(films.map{x => Rating(0, x._1, x._2)})
      }
  }
  ```
  
  - If ```interactive``` is set to false then we get the file stored in ```movielens-modified``` and create an RDD from it
  
- **Extra filtering:**
  In order to disregard movies that received few ratings, we record number of ratings for each movie, and then filter out those whose occurences are less than 50.
  ```scala
    var ratingsData0 = sc.textFile(path).map{ line =>
      val fields = line.split(",") #split by comma
      (fields(1).toInt, (fields(0).toInt, fields(2).toDouble))
    }
    val ratingsCount = ratingsData0.countByKey()
    val ratingsData = ratingsData0.filter(x => ratingsCount(x._1) > 49).map(x => Rating(x._2._1, x._1, x._2._2))
  ```
  - `(fields(1).toInt, (fields(0).toInt, fields(2).toDouble))` is a tuple of `(movie_id, (user_id, rating))`
  - Then for each movie in `ratingsData0` we count the number of ratings it has and store it as tuple `(movie_id, #_of_ratings)`
  -  `.filter(x => ratingsCount(x._1) > 49)` will iterate over ratings dataset and allow only movies that have number of ratings more than 49. 
  - Then `ratingsData` is split to train and test sets


## Running on local cluster <a id="rlc"> </a>
 - To run on the cluster we need to to execute the new command:
 ```
 park-submit --master yarn --conf spark.network.timeout=800 --class MovieLensALS shared/Assignment/spark-recommendation/target/scala-2.12/spark-recommendation_2.12-3.2.0_0.1.jar movielens-modified -user true true
 ```
 - `spark.network.timeout=800` is a command to allow spark to wait for nodes more than 120s, because the load is quite high and default value is not enough
 - The last two options `true true` tell the program to predict the ratings in an interactive manner by asking questions
 - The program asks a user to input preferences
 <a href="https://drive.google.com/uc?export=view&id=<FILEID>"><img src="https://drive.google.com/uc?export=view&id=1pnlWVqat2zCMw7D8yIC7ymMP2uO_qYik" style="width: 650px; max-width: 80%; height: auto" title="Click to enlarge picture"></a>
 - The program ouputs the predicted ratings:
 <a href="https://drive.google.com/uc?export=view&id=<FILEID>"><img src="https://drive.google.com/uc?export=view&id=1E1QwhTWWhCrFjjsvhb6wb3j7ATm8PMxK" style="width: 650px; max-width: 80%; height: auto" title="Click to enlarge picture"></a>
 - Ratings are being saved at `user_ratings.tsv`
 
 
### Video Demonstation (Interactive mode) <a id="vd"> </a>

<a href="https://drive.google.com/uc?export=view&id=<FILEID>"><img src="https://drive.google.com/uc?export=view&id=1dqSg-rLvtGiy_atpguOd8v5Ien-wG32I" style="width: 650px; max-width: 80%; height: auto" title="Click to enlarge picture"></a>

### Preparation of distributed cluster <a id="pdc"> </a>
- To run on distributed cluster we need to connect 2 nodes (Linux and Windows) to the same network and start 2 vagrant virtual machines (hadoop_image.box).
- It is crucial to add a following line in a vagrant file
``` config.vm.network "public_network", ip: "<IP>"```
That ip should be within network range, for us it was:
  - 192.168.49.15 
  - 192.168.49.17
- Configs for both machines:
<a href="https://drive.google.com/uc?export=view&id=<FILEID>"><img src="https://drive.google.com/uc?export=view&id=1US6jRkn-CM-jpFyqekAubgDWjBoUhXuT" style="width: 650px; max-width: 80%; height: auto" title="Click to enlarge picture"></a>
<a href="https://drive.google.com/uc?export=view&id=<FILEID>"><img src="https://drive.google.com/uc?export=view&id=1FtmiIrj3Ydv5tD0ONzcz15eVPMW5MV9A" style="width: 650px; max-width: 80%; height: auto" title="Click to enlarge picture"></a>
- Pinging both machines
<a href="https://drive.google.com/uc?export=view&id=<FILEID>"><img src="https://drive.google.com/uc?export=view&id=1qLWgbDQxjYR3-4Xy2h16iLMnS9rlmg4U" style="width: 650px; max-width: 80%; height: auto" title="Click to enlarge picture"></a>
<a href="https://drive.google.com/uc?export=view&id=<FILEID>"><img src="https://drive.google.com/uc?export=view&id=1nvLbwh-CwpNfxw54yq2fGIVKzWyn3XVt" style="width: 650px; max-width: 80%; height: auto" title="Click to enlarge picture"></a>
- Distribute the keys on both machines (first is namenode)
 <a href="https://drive.google.com/uc?export=view&id=<FILEID>"><img src="https://drive.google.com/uc?export=view&id=1m23Shl2H_MSI1JfXpF39b5ueE_TKZkeP" style="width: 650px; max-width: 80%; height: auto" title="Click to enlarge picture"></a>
 <a href="https://drive.google.com/uc?export=view&id=<FILEID>"><img src="https://drive.google.com/uc?export=view&id=1YqVMzr2Nd8QtnlOc0Ilhrdy4WbhldLav" style="width: 650px; max-width: 80%; height: auto" title="Click to enlarge picture"></a> 
- Now machines are able to ssh to each other
 <a href="https://drive.google.com/uc?export=view&id=<FILEID>"><img src="https://drive.google.com/uc?export=view&id=1xNl3FzzbX9KD1f4S1JToFhOep3PFzN7F" style="width: 650px; max-width: 80%; height: auto" title="Click to enlarge picture"></a> 
  <a href="https://drive.google.com/uc?export=view&id=<FILEID>"><img src="https://drive.google.com/uc?export=view&id=1U6ZbM6pULMLU9_6xGWAQ9q40FLoSAvcD" style="width: 650px; max-width: 80%; height: auto" title="Click to enlarge picture"></a> 
- Set up hadoop cluster in a similar way as with local cluster
- YARN should be configured as follows before proceeding
<a href="https://drive.google.com/uc?export=view&id=<FILEID>"><img src="https://drive.google.com/uc?export=view&id=1VidNtYIyiGOwjFuvQ4huZ9_FK4PCDNSX" style="width: 650px; max-width: 80%; height: auto" title="Click to enlarge picture"></a> 

## Running on distibuted cluster <a id="rdc"> </a>
For fast execution purposes we run the model on the subset of training data. Even that, we still ran into virtual memory exceed, which was fixed by adding in `yarn-site.xml`
```
<property>
    <name>yarn.nodemanager.vmem-check-enabled</name>
    <value>false</value>
    <description>Whether virtual memory limits will be enforced for containers</description>
</property>
```
- Run command as in local cluster
```
park-submit --master yarn --conf spark.network.timeout=800 --class MovieLensALS <local path to jar> <path on hdfs> -user <grade?> <ask interactively?>
```
- Interactive mode ask for user answers and stores every answer in created `user_ratings.tsv`
<a href="https://drive.google.com/uc?export=view&id=<FILEID>"><img src="https://drive.google.com/uc?export=view&id=1aLcIsdGJE3VeNGDlgJ6Dnacifiao659y" style="width: 650px; max-width: 80%; height: auto" title="Click to enlarge picture"></a> 
- And a `user_ratings.tsv` file
 <a href="https://drive.google.com/uc?export=view&id=<FILEID>"><img src="https://drive.google.com/uc?export=view&id=1erFu2muMDEOC508dy4pay6_VEfhAelHB" style="width: 650px; max-width: 80%; height: auto" title="Click to enlarge picture"></a> 
- Running with a non-interactive mode
 <a href="https://drive.google.com/uc?export=view&id=<FILEID>"><img src="https://drive.google.com/uc?export=view&id=14uObmnri5MJmn-ac9PZYGxRBzeIb44rt" style="width: 650px; max-width: 80%; height: auto" title="Click to enlarge picture"></a> 

## Results <a id="res"> </a>

- Finished the code
- Compiled the scala code with sbt
- Ran the program on local cluster
- Ran the program on distributed cluster

## Discussion on model (Ratings of Scala) <a id="dm"> </a>

Here is the graph of how model error is affected by rank
 <a href="https://drive.google.com/uc?export=view&id=<FILEID>"><img src="https://drive.google.com/uc?export=view&id=12V9YNcP74EpH0REAgM16Xt7vfvEXjxZ_" style="width: 650px; max-width: 80%; height: auto" title="Click to enlarge picture"></a> 
 
So we can observe that the best rank is 8, lower or higher rank results in a sligtly worser error. Perhaps this is due to model starting overfitting after some point.


## Contacts & Contribution

Team name: **2**

* Vladimir Bliznyukov ([@univacone](https://t.me/univacone)), 
    * completed code
    * deployed the system on cluster
    * wrote report
* Parth Kalkar ([@ParthKalkar](https://t.me/ParthKalkar)),
    * created github repository
    * configured distributed cluster
    * wrote report
* Violetta Sim ([@eyihluyc](https://t.me/eyihluyc))
    * completed code
    * configured distributed cluster
    * wrote report
    
## Self-Check Questions  <a id="sq"> </a>
1. What does sc.parallelize do?
    - The `sc. parallelize()` creates a parallelized collection, which allows Spark context to distribute the load across multiple nodes, instead of running everything on a single node to process the data. That way task the tasks are being performed concurrently. 

2. What does collectAsMap do?

    - collectAsMap() will return the results for key-value paired RDD as Map collection. Since it is returning Map collection we will only get pairs with unique keys. Thus pairs with duplicate keys will be removed.
3. What is the difference between foreach and map?
    As you've noticed already, the map function in the Seq trait returns a value. Its signature in fact is

    - def map[B](f: (A) ⇒ B): Seq[B] (Return a new mapped sequence) map is designed to apply a function to every element of a collection extending the Seq trait and return a new collection.
   -  def foreach(f: (A) ⇒ Unit): Unit (Perform an action, nothing returned)

    
4. What is pattern matching in Scala?
    - Pattern matching is a way of checking the given sequence of tokens for the presence of the specific pattern. 
    - “**match**” keyword is used instead of switch statement (has alternatives of matching) 
    - For eg. 
    ```scala=
    // the pattern matching

    object patternMatching {

        // main method
        def main(args: Array[String]) {

        // calling test method
        println(test("Student"));
        }


    // method containing match keyword
    def test(x:String): String = x match {

        // if value of x is "G1",
        // this case will be executed
        case "C1" => "STD"

        // if value of x is "G2",
        // this case will be executed
        case "C2" => "Teachers"

        // if x doesnt match any sequence,
        // then this case will be executed
        case _ => "Default Case Executed"
    }
    }
    ```
   
    
## Additional Material  <a id="am"> </a>

### Collaborative filtering

Collaborative filtering is commonly used for recommender systems. These techniques aim to fill in the missing entries of a user-item association matrix.  *spark.ml* uses the alternating least squares (ALS) algorithm to learn these factors. The implementation in *spark.ml* has the following parameters:

1. *numBlocks* is the number of blocks the users and items will be partitioned into in order to parallelize computation (defaults to 10).

2. *rank* is the number of latent factors in the model (defaults to 10).

3. *maxIter* is the maximum number of iterations to run (defaults to 10).

4. *regParam* specifies the regularization parameter in ALS (defaults to 1.0).

5. *implicitPrefs* specifies whether to use the explicit feedback ALS variant or one adapted for implicit feedback data (defaults to **false** which means using explicit feedback).

6. *alpha* is a parameter applicable to the implicit feedback variant of ALS that governs the baseline confidence in preference observations (defaults to 1.0).

7. *nonnegative* specifies whether or not to use nonnegative constraints for least squares (defaults to false).


### Alternating Least Squares(ALS) 
The alternating least squares (ALS) algorithm factorizes a given matrix R(rating_matrix) into two factors U(user_matrix) and V(item_matrix) such that R≈(U_tranpose)V. The unknown row dimension is given as a parameter to the algorithm and is called latent factors.

In order to find the user and item matrix, the following problem is solved:

![](https://i.imgur.com/Zd8jYTp.png)

By fixing one of the matrices U or V, we obtain a quadratic form which can be solved directly. The solution of the modified problem is guaranteed to monotonically decrease the overall cost function. By applying this step alternately to the matrices U and V, we can iteratively improve the matrix factorization.

The matrix R is given in its sparse representation as a tuple of (i,j,r) where *i* denotes the row index, *j* the column index and *r* is the matrix value at position (i,j).


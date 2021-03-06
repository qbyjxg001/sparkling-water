# Sparkling Water Meetup 

## Download

Please download [Sparkling Water
0.2.1-58](http://h2o-release.s3.amazonaws.com/sparkling-water/master/58/index.html) and unzip the file:
```
unzip sparkling-water-0.2.1-58.zip
cd sparkling-water-0.2.1-58
```

## Step-by-Step through Airlines with Weather Data Example

1. Run Sparkling shell with an embedded cluster:
  ```
  export SPARK_HOME="/path/to/spark/installation"
  export MASTER="local-cluster[3,2,1024]"
  bin/sparkling-shell
  ```

2. You can go to [http://localhost:4040/](http://localhost:4040/) to see the Sparkling shell (i.e., Spark driver) status.

3. Create H<sub>2</sub>O cloud using all 3 Spark workers
  ```scala
  import org.apache.spark.h2o._
  import org.apache.spark.examples.h2o._
  val h2oContext = new H2OContext(sc).start()
  import h2oContext._
  ```

4. Load weather data for Chicago international airport (ORD) with help of RDD API.
  ```scala
  val weatherDataFile = "examples/smalldata/Chicago_Ohare_International_Airport.csv"
  val wrawdata = sc.textFile(weatherDataFile,3).cache()
  val weatherTable = wrawdata.map(_.split(",")).map(row => WeatherParse(row)).filter(!_.isWrongRow())
  ```

5. Load airlines data using H<sub>2</sub>O parser
  ```scala
  import java.io.File
  val dataFile = "examples/smalldata/year2005.csv.gz"
  val airlinesData = new DataFrame(new File(dataFile))
  ```

6. Select flights with destination in Chicago (ORD)
  ```scala
  val airlinesTable : RDD[Airlines] = toRDD[Airlines](airlinesData)
  val flightsToORD = airlinesTable.filter(f => f.Dest==Some("ORD"))
  ```
  
7. Compute number of these flights
  ```scala
  flightsToORD.count
  ```

8. Use Spark SQL to join flight data with weather data
  ```scala
  import org.apache.spark.sql.SQLContext
  val sqlContext = new SQLContext(sc)
  import sqlContext._
  flightsToORD.registerTempTable("FlightsToORD")
  weatherTable.registerTempTable("WeatherORD")
  ```

9. Perform SQL JOIN on both tables
  ```scala
  val joinedTable = sql(
          """SELECT
            |f.Year,f.Month,f.DayofMonth,
            |f.CRSDepTime,f.CRSArrTime,f.CRSElapsedTime,
            |f.UniqueCarrier,f.FlightNum,f.TailNum,
            |f.Origin,f.Distance,
            |w.TmaxF,w.TminF,w.TmeanF,w.PrcpIn,w.SnowIn,w.CDD,w.HDD,w.GDD,
            |f.ArrDelay
            |FROM FlightsToORD f
            |JOIN WeatherORD w
            |ON f.Year=w.Year AND f.Month=w.Month AND f.DayofMonth=w.Day""".stripMargin)
  ```
  
10. Split data into train/validation/test datasets
  ```scala
  import hex.splitframe.SplitFrame
  import hex.splitframe.SplitFrameModel.SplitFrameParameters

  val sfParams = new SplitFrameParameters()
  sfParams._train = joinedTable
  sfParams._ratios = Array(0.7, 0.2)
  val sf = new SplitFrame(sfParams)

  val splits = sf.trainModel().get._output._splits
  val trainTable = splits(0)
  val validTable = splits(1)
  val testTable  = splits(2)
  ```
  
11. Run deep learning to produce model estimating arrival delay:
  ```scala
  import hex.deeplearning.DeepLearning
  import hex.deeplearning.DeepLearningModel.DeepLearningParameters
  val dlParams = new DeepLearningParameters()
  dlParams._train = trainTable
  dlParams._response_column = 'ArrDelay
  dlParams._valid = validTable
  dlParams._epochs = 100
  dlParams._reproducible = false
  dlParams._force_load_balance = false

  // Invoke model training
  val dl = new DeepLearning(dlParams)
  val dlModel = dl.trainModel.get
  ```

12. Use model to estimate delay on training data
  ```scala
  val dlPredictTable = dlModel.score(testTable)('predict)
  val predictionsFromDlModel = toRDD[DoubleHolder](dlPredictTable).collect.map(_.result.getOrElse(Double.NaN))
  ```

  
13. Generate R code to plot residuals plot
  ```scala
  import org.apache.spark.examples.h2o.DemoUtils.residualPlotRCode
  residualPlotRCode(dlPredictTable, 'predict, testTable, 'ArrDelay)
  ```
  
  Use resulting R code inside RStudio:
  ```R
  #
  # R script for residual plot
  #
  # Import H2O library
  library(h2o)
  # Initialize H2O R-client
  h = h2o.init()
  # Fetch prediction and actual data, use remembered keys
  pred = h2o.getFrame(h, "dframe_b5f449d0c04ee75fda1b9bc865b14a69")
  act = h2o.getFrame (h, "frame_rdd_14_b429e8b43d2d8c02899ccb61b72c4e57")
  # Select right columns
  predDelay = pred$predict
  actDelay = act$ArrDelay
  # Make sure that number of rows is same  
  nrow(actDelay) == nrow(predDelay)
  # Compute residuals  
  residuals = predDelay - actDelay
  # Plot residuals   
  compare = cbind (as.data.frame(actDelay$ArrDelay), as.data.frame(residuals$predict))
  nrow(compare)
  plot( compare[,1:2] )
  ```
  
14. Try to generate better model with GBM
  ```scala
  import hex.tree.gbm.GBM
  import hex.tree.gbm.GBMModel.GBMParameters

  val gbmParams = new GBMParameters()
  gbmParams._train = trainTable
  gbmParams._response_column = 'ArrDelay
  gbmParams._valid = validTable
  gbmParams._ntrees = 100

  val gbm = new GBM(gbmParams)
  val gbmModel = gbm.trainModel.get
  ```
  
15. Make prediction and print R code for residual plot
  ```scala
  val gbmPredictTable = gbmModel.score(testTable)('predict)
  residualPlotRCode(gbmPredictTable, 'predict, testTable, 'ArrDelay)
  ```

  
  
   

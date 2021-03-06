Using Grid Search GBM in Spark Pipelines
----------------------------------------

H2O's Grid Search for GBM is exposed for the Spark pipelines. This tutorial demonstrates how it is used in a simple Spark pipeline on the Ham or Spam dataset.

Prepare the environment
~~~~~~~~~~~~~~~~~~~~~~~

Add the data to Spark:

.. code:: scala

    import org.apache.spark.SparkFiles
    import water.support.SparkContextSupport
    SparkContextSupport.addFiles(sc, "/path/to/smsData.txt")


Prepare the method for loading the data:

.. code:: scala

    import org.apache.spark.sql.types.{StringType, StructField, StructType}
    import org.apache.spark.sql.{DataFrame, Row, SQLContext}

    implicit val sqlContext = spark.sqlContext

    def load(dataFile: String)(implicit sqlContext: SQLContext): DataFrame = {
      val smsSchema = StructType(Array(
        StructField("label", StringType, nullable = false),
        StructField("text", StringType, nullable = false)))
      val rowRDD = sc.textFile(SparkFiles.get(dataFile)).map(_.split("\t", 2)).filter(r => !r(0).isEmpty).map(p => Row(p(0),p(1)))
      sqlContext.createDataFrame(rowRDD, smsSchema)
    }


Make sure ``H2OContext`` is available:

.. code:: scala

    import org.apache.spark.h2o._
    implicit val h2oContext = H2OContext.getOrCreate(spark)


Define the Pipeline Stages
~~~~~~~~~~~~~~~~~~~~~~~~~~

Tokenize the Messages
#####################

This Spark Transformer tokenizes the messages and splits sentences into words.

.. code:: scala

    val tokenizer = new RegexTokenizer().
      setInputCol("text").
      setOutputCol("words").
      setMinTokenLength(3).
      setGaps(false).
      setPattern("[a-zA-Z]+")

Remove Ignored Words
####################

Remove words that do not bring much value for the model.

.. code:: scala

    val stopWordsRemover = new StopWordsRemover().
      setInputCol(tokenizer.getOutputCol).
      setOutputCol("filtered").
      setStopWords(Array("the", "a", "", "in", "on", "at", "as", "not", "for")).
      setCaseSensitive(false)

Hash the Words
##############

Crete hashes for the observed words.

.. code:: scala

    val hashingTF = new HashingTF().
      setNumFeatures(1 << 10).
      setInputCol(stopWordsRemover.getOutputCol).
      setOutputCol("wordToIndex")

Create an Inverse Document Frequencies Model
############################################

Create an IDF model. This creates a numerical representation of how much information a given word provides in the whole message.

.. code:: scala

    val idf = new IDF().
      setMinDocFreq(4).
      setInputCol(hashingTF.getOutputCol).
      setOutputCol("tf_idf")

Create a Grid Search GBM Model
##############################

First, we need to define the hyper parameters. Hyper parameters are stored in the map where key is the name of the parameter and value is an array of possible values.

We also need to specify the algorithm on which we want to run Grid Search together with its arguments. For this, we can use ``setAlgo`` method.

.. code:: scala

    import scala.collection.mutable.HashMap
    import org.apache.spark.ml.h2o.algos.{H2OGBM, H2OGridSearch}

    val hyperParams: HashMap[String, Array[AnyRef]] = HashMap()
    hyperParams += ("_ntrees" -> Array(1, 30).map(_.asInstanceOf[AnyRef]))

    val grid = new H2OGridSearch().
      setLabelCol("label").
      setHyperParameters(hyperParams).
      setAlgo(new H2OGBM().setMaxDepth(30))

Remove Temporary Columns
########################

Remove unnecessary columns:

.. code:: scala

    val colPruner = new ColumnPruner().
      setColumns(Array[String](idf.getOutputCol, hashingTF.getOutputCol, stopWordsRemover.getOutputCol, tokenizer.getOutputCol))

Create and Train the Pipeline
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: scala

    val pipeline = new Pipeline().
      setStages(Array(tokenizer, stopWordsRemover, hashingTF, idf, grid, colPruner))

    // Train the pipeline model
    val data = load("smsData.txt")
    val model = pipeline.fit(data)

If you are interested in what hyper-parameter *("_ntrees")* value got selected for the best model, get a given stage from
the pipeline model and cast it to ``H2OTreeBasedSupervisedMOJOModel``. *This statement is relevant only to tree-based
algorithms like GBM, DRF and XGBoost.*

.. code:: scala

    val bestH2OModel = model.stages(4).asInstanceOf[H2OTreeBasedSupervisedMOJOModel]
    println(s"_ntrees value: ${bestH2OModel.getNtrees()}")


Run Predictions
~~~~~~~~~~~~~~~

Prepare the predictor function:

.. code:: scala

    def isSpam(smsText: String,
               model: PipelineModel,
               hamThreshold: Double = 0.5) = {
      val smsTextSchema = StructType(Array(StructField("text", StringType, nullable = false)))
      val smsTextRowRDD = sc.parallelize(Seq(smsText)).map(Row(_))
      val smsTextDF = sqlContext.createDataFrame(smsTextRowRDD, smsTextSchema)
      val prediction = model.transform(smsTextDF)
      prediction.select("prediction.p1").first.getDouble(0) > hamThreshold
    }

And finally, run the predictions:

.. code:: scala

    println(isSpam("Michal, h2oworld party tonight in MV?", model))

    println(isSpam("We tried to contact you re your reply to our offer of a Video Handset? 750 anytime any networks mins? UNLIMITED TEXT?", model))

### Basic concepts: Estimator/Transformer In MLSQL


There are three kinds of data processing model which are described in following picture:

![](https://github.com/allwefantasy/streamingpro/raw/master/images/WX20181111-201226@2x.png)


The 'BlackBox' we also call them Estimator or Transformer.

If the BlackBox generate some knowledge(we call model) after the data been processed, then we call the
BlackBox Estimator otherwise we call it Transformer. The knowledge(model) is also a transformer since it can predict data
without new model created.

In Machine learning field, the Estimator is also know as Algorithm, and the model generated by Estimator is called transformer.

In MLSQL, there are train/run/predict statements to fit these concepts.

For example:

```sql
train data as RandomForest.`/tmp/model`
where maxDepth="10";
```
The magic thing is the estimator `RandomForest`, this estimator will create a model
after processing data.

From this example, you will find you can do any data process with custom estimator/transformer.

In next section, we will tech you how to do that.

### Tutorial : How to develop a RandomForest estimator


A estimator at least contains:

1. How to train data
2. How to batch predict data
3. How to convert model to a UDF

and some options:

1. the doc
2. the example
3. Compatibility of core engine(Spark)
4. the params available
5. the value of every param in model
6. the estimator type: normal data processing or algorithm

When you extends `streaming.dsl.mmlib.SQLAlg` you need implements at least several methods:

```scala
def train(df: DataFrame, path: String, params: Map[String, String]): DataFrame

def load(sparkSession: SparkSession, path: String, params: Map[String, String]): Any

def predict(sparkSession: SparkSession, _model: Any, name: String, params: Map[String, String]): UserDefinedFunction

def batchPredict(df: DataFrame, path: String, params: Map[String, String]): DataFrame
```

In the train method , the data is the first parameter df, and the model generated by this estimator should be saved in the
second parameter path. Also you can get the params configured in where conditions from the third
parameter params. If necessary, you should return a dataframe show some meta data e.g. where we saved the model, the train is whether successful.

Once we have generate the model, we will  register it as a function:

```sql
register RandomForest.`/tmp/model` as rf_predict.
```

When the register statement is executed, then the load and predict will be called.
load method will load the model and then call predict, predict method will generate a UDF which can be used in
SQL.

batchPredict is simple, we give you DataFrame and the model path, you should transform the data and return new DataFrame.

The good news is that we do not do all these from scratch, MLSQL have already provide us a lot of tools.

Now let's take the journey.

### Train

Suppose you have created a class named SQLRandomForest.

```scala
class SQLRandomForest extends SQLAlg  {
```

then the IDE will generate some methods need to implemented automatically. The first is train method.

RandomForest is a classification algorithm, so we need some basic params e.g.keepVersion,evaluateTable.

We can get these help from `streaming.dsl.mmlib.algs.classfication.BaseClassification`,  Since we use scala, just mixin the trait.

Now ,the class should look like this:

```scala
class SQLRandomForest(override val uid: String) extends SQLAlg with BaseClassification {

  def this() = this(BaseParams.randomUID())

  override def train(df: DataFrame, path: String, params: Map[String, String]): DataFrame = {

```

The first step is configure the param from where clause and set them to parameters in SQLRandomForest.

```scala
val keepVersion = params.getOrElse("keepVersion", "true").toBoolean
setKeepVersion(keepVersion)

// or you can do like this:

params.get(keepVersion.name).
      map(m => set(keepVersion, m.toBoolean)).
      getOrElse($(keepVersion))

```

the keepVersion is in `streaming.dsl.mmlib.algs.param.BaseParams` which is look like this:

```scala
final val keepVersion: BooleanParam = new BooleanParam(this, "keepVersion", "If set true, then every time you run the " +
    "algorithm, it will generate a new directory to save the model.")

  setDefault(keepVersion -> true)

  final def getKeepVersion: Boolean = $(keepVersion)

  def setKeepVersion(value: Boolean): this.type = set(keepVersion, value)
```

If we set keepVersion to true ,that means every time we run train statement, we should create
a new directory to save the model generated by the SQLRandomForest otherwise the model should
be overwrite every time.

The next step is check we should increment the version, if we have set keepVersion to true:

```scala
SQLPythonFunc.incrementVersion(path, keepVersion)
val spark = df.sparkSession
```

In most case, estimator should support train  in parallel with different set of  parameters.

```sql
train data1 as RandomForest.`/tmp/model` where

-- once set true,every time you run this script, MLSQL will generate new directory for you model
keepVersion="true"

-- specify the test dataset which will be used to feed evaluator to generate some metrics e.g. F1, Accurate
and evaluateTable="data1"

-- specify group 0 parameters
and `fitParam.0.labelCol`="features"
and `fitParam.0.featuresCol`="label"
and `fitParam.0.maxDepth`="2"

-- specify group 1 parameters
and `fitParam.1.featuresCol`="features"
and `fitParam.1.labelCol`="label"
and `fitParam.1.maxDepth`="10"
```

In the above code, we have configured group 0 and group 1. They have different parameters are set.
To implement this, you can mixin `streaming.dsl.mmlib.algs.Functions`, now the class should like this:

```scala
class SQLRandomForest(override val uid: String) extends SQLAlg with Functions with BaseClassification {
```


There is a method called `trainModelsWithMultiParamGroup` in Functions, you can call this method directly in
train method.

```scala
trainModelsWithMultiParamGroup[RandomForestClassificationModel](df, path, params, () => {
      new RandomForestClassifier()
    }, (_model, fitParam) => {
      evaluateTable match {
        case Some(etable) =>
          val model = _model.asInstanceOf[RandomForestClassificationModel]
          val evaluateTableDF = spark.table(etable)
          val predictions = model.transform(evaluateTableDF)
          multiclassClassificationEvaluate(predictions, (evaluator) => {
            evaluator.setLabelCol(fitParam.getOrElse("labelCol", "label"))
            evaluator.setPredictionCol("prediction")
          })

        case None => List()
      }
    }
    )
```

trainModelsWithMultiParamGroup needs you tell him the model type will be generated and the data,path, params and
the inner estimator.

Here, we find that there is a estimator have been implemented by Spark MLLib, so we just
new a RandomForestClassifier.

The second function in trainModelsWithMultiParamGroup is that we should tell  trainModelsWithMultiParamGroup how
we deal with evaluate the model when the user provide evaluateTable.

This is the only thing we should do, and the other thing include how to save the RandomForestClassificationModel will
be automatically implemented by trainModelsWithMultiParamGroup.

finally, we can output some message to user when we have finished our train stage:

```scala
formatOutput(getModelMetaData(spark, path))
```

the formatOutput is from `streaming.dsl.mmlib.algs.MllibFunctions`, so let's mixin it too.


```scala
class SQLRandomForest(override val uid: String) extends SQLAlg with MllibFunctions with Functions with BaseClassification {

  def this() = this(BaseParams.randomUID())

  override def train(df: DataFrame, path: String, params: Map[String, String]): DataFrame = {


    val keepVersion = params.getOrElse("keepVersion", "true").toBoolean
    setKeepVersion(keepVersion)

    val evaluateTable = params.get("evaluateTable")
    setEvaluateTable(evaluateTable.getOrElse("None"))

    SQLPythonFunc.incrementVersion(path, keepVersion)
    val spark = df.sparkSession

    trainModelsWithMultiParamGroup[RandomForestClassificationModel](df, path, params, () => {
      new RandomForestClassifier()
    }, (_model, fitParam) => {
      evaluateTable match {
        case Some(etable) =>
          val model = _model.asInstanceOf[RandomForestClassificationModel]
          val evaluateTableDF = spark.table(etable)
          val predictions = model.transform(evaluateTableDF)
          multiclassClassificationEvaluate(predictions, (evaluator) => {
            evaluator.setLabelCol(fitParam.getOrElse("labelCol", "label"))
            evaluator.setPredictionCol("prediction")
          })

        case None => List()
      }
    }
    )

    formatOutput(getModelMetaData(spark, path))
  }
```

### Batch Predict

The first step is to load the model saved in preview stage. So we should implements load method:

```scala
override def load(sparkSession: SparkSession, path: String, params: Map[String, String]): Any = {

    val (bestModelPath, baseModelPath, metaPath) = mllibModelAndMetaPath(path, params, sparkSession)
    val model = RandomForestClassificationModel.load(bestModelPath(0))
    ArrayBuffer(model)
  }
```

There is a function `mllibModelAndMetaPath` will find the model paths automatically for you.
The return value is bestModelPath, baseModelPath,metaPath. mllibModelAndMetaPath
will choose the  model paths  sorted by the metric computed in train stage( remember the evaluateTable in train method?).
If there is no metric, then the first one will be return. It  also support manually specify the model by algIndex.

```sql
-- manually specify the model
predict data RandomForest.`/tmp/model` where algIndex="0";
```

Then ,you can load the path and get the real model with this:

```scala
val model = RandomForestClassificationModel.load(bestModelPath(0))
```

and finally return `ArrayBuffer(model)`.

Assume you have already load the model, now we can batch predict data:

```scala
override def batchPredict(df: DataFrame, path: String, params: Map[String, String]): DataFrame = {
    val model = load(df.sparkSession, path, params).asInstanceOf[ArrayBuffer[RandomForestClassificationModel]].head
    model.transform(df)
  }
```

We know that the mode is `RandomForestClassificationModel`, and this model have transform function which can be
apply to DataFrame directly, so we just invoke the transform and return the result.

### Register a model as UDF

In preview chapter, we have use load method to load model and wrap with ArrayBuffer,
This step the target is we convert the model to a Spark SQL UDF. Since RandomForest is
a classification algorithm, so there is a pretty fit method called predict_classification will
do this job:

```scala
override def predict(sparkSession: SparkSession, _model: Any, name: String, params: Map[String, String]): UserDefinedFunction = {
    predict_classification(sparkSession, _model, name)
  }
```

Here the variable `name` is udf name correspond to the `rf_predict` in register statement:

```sql
register RandomForest.`/tmp/model` as rf_predict.
```

Here is the detail of predict_classification:

```scala
def predict_classification(sparkSession: SparkSession, _model: Any, name: String) = {

    val models = sparkSession.sparkContext.broadcast(_model.asInstanceOf[ArrayBuffer[Any]])

    val raw2probabilityMethod = if (sparkSession.version.startsWith("2.3")) "raw2probabilityInPlace" else "raw2probability"

    val f = (vec: Vector) => {
      models.value.map { model =>
        val predictRaw = model.getClass.getMethod("predictRaw", classOf[Vector]).invoke(model, vec).asInstanceOf[Vector]
        val raw2probability = model.getClass.getMethod(raw2probabilityMethod, classOf[Vector]).invoke(model, predictRaw).asInstanceOf[Vector]
        //model.getClass.getMethod("probability2prediction", classOf[Vector]).invoke(model, raw2probability).asInstanceOf[Vector]
        //概率，分类
        (raw2probability(raw2probability.argmax), raw2probability)
      }.sortBy(f => f._1).reverse.head._2
    }

    val f2 = (vec: Vector) => {
      models.value.map { model =>
        val predictRaw = model.getClass.getMethod("predictRaw", classOf[Vector]).invoke(model, vec).asInstanceOf[Vector]
        val raw2probability = model.getClass.getMethod(raw2probabilityMethod, classOf[Vector]).invoke(model, predictRaw).asInstanceOf[Vector]
        //model.getClass.getMethod("probability2prediction", classOf[Vector]).invoke(model, raw2probability).asInstanceOf[Vector]
        raw2probability
      }
    }

    sparkSession.udf.register(name + "_raw", f2)

    UserDefinedFunction(f, VectorType, Some(Seq(VectorType)))
  }
```

The first step is broadcast the model, the second step is create function(UDF)
and we get the model broadcast and invoke the predict method in the model by reflection.

## params

As we know , we can use where clause to configure our estimator/transformer. If you want, you can use load statement to get all
these messages:

```sql
load modelParams.`RandomForest` as output;
```

modelParams will invoke `explainParams` in SQLRandomForest to get information of params. Here is the code
we implements:

```sql
 override def explainParams(sparkSession: SparkSession): DataFrame = {
    _explainParams(sparkSession, () => {
      new RandomForestClassifier()
    })
  }
```

_explainParams is from the supper class, and it will combine the params from RandomForestClassifier and params from SQLRandomForest.
The params from  SQLRandomForest contains keepVersion,evaluateTable.

## Doc and Example

Any Estimator/Transformer should have Doc and Example. Please implements the doc/codeExample method.

```scala
override def doc: Doc = Doc(HtmlDoc,
    """
      | <a href="http://en.wikipedia.org/wiki/Random_forest">Random Forest</a> learning algorithm for
      | classification.
      | It supports both binary and multiclass labels, as well as both continuous and categorical
      | features.
      |
      | Use "load modelParams.`RandomForest` as output;"
      |
      | to check the available hyper parameters;
      |
      | Use "load modelExample.`RandomForest` as output;"
      | get example.
      |
      | If you wanna check the params of model you have trained, use this command:
      |
      | ```
      | load modelExplain.`/tmp/model` where alg="RandomForest" as outout;
      | ```
      |
    """.stripMargin)


  override def codeExample: Code = Code(SQLCode, CodeExampleText.jsonStr +
    """
      |load jsonStr.`jsonStr` as data;
      |select vec_dense(features) as features ,label as label from data
      |as data1;
      |
      |-- use RandomForest
      |train data1 as RandomForest.`/tmp/model` where
      |
      |-- once set true,every time you run this script, MLSQL will generate new directory for you model
      |keepVersion="true"
      |
      |-- specify the test dataset which will be used to feed evaluator to generate some metrics e.g. F1, Accurate
      |and evaluateTable="data1"
      |
      |-- specify group 0 parameters
      |and `fitParam.0.labelCol`="features"
      |and `fitParam.0.featuresCol`="label"
      |and `fitParam.0.maxDepth`="2"
      |
      |-- specify group 1 parameters
      |and `fitParam.1.featuresCol`="features"
      |and `fitParam.1.labelCol`="label"
      |and `fitParam.1.maxDepth`="10"
      |;
    """.stripMargin)
```



## explainModel

Once we have generated a model, sometimes people may want to know treeWeights of the RandomForest. The explainModel
will be invoked to provide the information:

```scala
override def explainModel(sparkSession: SparkSession, path: String, params: Map[String, String]): DataFrame = {
    val models = load(sparkSession, path, params).asInstanceOf[ArrayBuffer[RandomForestClassificationModel]]
    val rows = models.flatMap { model =>
      val modelParams = model.params.filter(param => model.isSet(param)).map { param =>
        val tmp = model.get(param).get
        val str = if (tmp == null) {
          "null"
        } else tmp.toString
        Seq(("fitParam.[group]." + param.name), str)
      }
      Seq(
        Seq("uid", model.uid),
        Seq("numFeatures", model.numFeatures.toString),
        Seq("numClasses", model.numClasses.toString),
        Seq("numTrees", model.treeWeights.length.toString),
        Seq("treeWeights", model.treeWeights.mkString(","))
      ) ++ modelParams
    }.map(Row.fromSeq(_))
    sparkSession.createDataFrame(sparkSession.sparkContext.parallelize(rows, 1),
      StructType(Seq(StructField("name", StringType), StructField("value", StringType))))
  }
```

The first step is to load the model, and then we will create DataFame contains fields  name and value.





















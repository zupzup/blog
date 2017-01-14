Machine Learning gets lots of attention all around 

Intro:
ML is fun and can be useful, even for smaller problems. We will look at Logistic Regression and use it to classify students depending on their exam scores if they are going to be admitted. (contrived example, but the same principles work for other things as well)

Short Primer on Logistic Regression - What is it used for? Regression algorithm, used for classification, think images of cancers and classify as benign or malignant depending on the input data (pixels). Won't go into the nitty gritty, but here are some resources and examples. Basic idea: we have input data and want to know in which category the thing falls depending on this data.

Outline Goal, get some understanding of what such a process could look like. We will use a small sample data set, so the results aren't interesting - this is not the goal, but just go through some of the many steps needed for a production ready ML application, to showcase some of it with Go.

Introduce goml, several libs (link them), but i chose goml for this.

## Data

First of all, we need some data. In this example, we use a very simple, csv-based testset of 

```go
xTrain, yTrain, err := base.LoadDataFromCSV("./data/studentsTrain.csv")
if err != nil {
    return err
}
xTest, yTest, err := base.LoadDataFromCSV("./data/studentsTest.csv")
if err != nil {
    return err
}
```

We can also create a scatter plot of the data to get a feel for it using [library1]()

```go
func plotData(xTest [][]float64, yTest []float64) error {
    p, err := plot.New()
    if err != nil {
        return err
    }
    p.Title.Text = "Exam Results"
    p.X.Label.Text = "X"
    p.Y.Label.Text = "Y"
    p.X.Max = 120
    p.Y.Max = 120

    positives := make(plotter.XYs, len(yTest))
    negatives := make(plotter.XYs, len(yTest))
    for i := range xTest {
        if yTest[i] == 1.0 {
            positives[i].X = xTest[i][0]
                positives[i].Y = xTest[i][1]
        }
        if yTest[i] == 0.0 {
            negatives[i].X = xTest[i][0]
                negatives[i].Y = xTest[i][1]
        }
    }

    err = plotutil.AddScatters(p, "Negatives", negatives, "Positives", positives)
    if err != nil {
        return err
    }
    if err := p.Save(10*vg.Inch, 10*vg.Inch, "exams.png"); err != nil {
        return err
    }
    return nil
}
```

We just take our `x` values to plot the different points and give them a color depending on their correspongin `y` value. This way we have a nice view where the positive and negative outcomes are and how they are distributed.

## Model

Learning the model is pretty simple using `goml`. We just call `linear.NewLogistic`, which takes the following parameters:

* Optimization Method
  * Explain
* Learning Rate
  * Explain
* Regularization
  * Explain
* Maximum Iterations
  * Explain
* Input Values (xTrain)
* Classification Attribute (yTrain)

```go
model := linear.NewLogistic(base.BatchGA, 0.00001, 0, 1000, xTrain, yTrain)
err := model.Learn()
if err != nil {
    return nil, nil, err
}
```

Alright, now we have our trained model - easy, right? With this newly created model, we can now use our model to predict the classification of new data using `model.Predict(input []float)`. For example, if we trained a model to classify pictures depending on whether there is a cat in them or not, the `Predict` method could, for a new image, tell us whether our model thinks there is a cat in it.

We can also use this prediction method to evaluate the performance of our trained model on the test set. For this purpose, we will create a so-called Confusion Matrix with the following data structure:

```go
// ConfusionMatrix describes a confusion matrix
type ConfusionMatrix struct {
    positive      int
    negative      int
    truePositive  int
    trueNegative  int
    falsePositive int
    falseNegative int
    accuracy      float64
}
```

//TODO: Explain Confusion Matrix

In order to implement this, we start by collecting all the positive and negative values of our testSet:

```go
cm := ConfusionMatrix{}
for _, y := range yTest {
    if y == 1.0 {
        cm.positive++
    }
    if y == 0.0 {
        cm.negative++
    }
}
```

Then, we iterate over the testSet, predict the outcome for each data point and record where in our confusion matrix the prediction lands. With these records, we can calculate the accuracy of our model:

```go
// Evaluate the Model on the Test data
for i := range xTest {
    prediction, err := model.Predict(xTest[i])
    if err != nil {
        return nil, nil, err
    }
    y := int(yTest[i])
    positive := prediction[0] >= decisionBoundary

   if y == 1 && positive {
       cm.truePositive++
   }
   if y == 1 && !positive {
       cm.falseNegative++
   }
   if y == 0 && positive {
       cm.falsePositive++
   }
   if y == 0 && !positive {
       cm.trueNegative++
   }
}

// Calculate Evaluation Metrics
cm.accuracy = float64(float64(cm.truePositive)+float64(cm.trueNegative)) / float64(float64(cm.positive)+float64(cm.negative))
```

Now that we are able to evaluate our model on the test data, we could actually try to tinker with the values to increase our accuracy even further. In practice, we would do this automated of course, testing different ranges of values for each relevant parameter, until we found the best ones. This also could be completely parallelized.
A very naive implementation without parallelization, which trys different values for the decision boundary could look like this: 

```go
var maxAccuracy float64
var maxAccuracyCM *ConfusionMatrix
var maxAccuracyDb float64
var maxAccuracyModel *linear.Logistic

//Try different parameters to get the best model
for db := 0.05; db < 1.0; db += 0.01 {
    // Learn Model and Calculate ConfusionMatrix for given values
    cm, model, err := tryValues(0.0001, 0.0, 1000, db, xTrain, xTest, yTrain, yTest)
    if err != nil {
        return err
    }
    if cm.accuracy > maxAccuracy {
        maxAccuracy = cm.accuracy
        maxAccuracyCM = cm
        maxAccuracyDb = db
        maxAccuracyModel = model
    }
}

fmt.Printf("Maximum accuracy: %.2f\n\n", maxAccuracy)
fmt.Printf("with Model: %s\n\n", maxAccuracyModel)
fmt.Printf("with Confusion Matrix:\n%s\n\n", maxAccuracyCM)
fmt.Printf("with Decisiion Boundary: %.2f\n", maxAccuracyDb)
fmt.Printf("with Num Iterations: %d\n", maxAccuracyIter)
```

We could to this for all values which are relevant to the model now until we found a model we're satisfied with. There are, of course, less brute-forceish methods of improving your models in the world of Machine Learning theory, which I won't cover here.

## Conclusion 

Conclusion

Of course only scratched the surface, but a fun little example where you can tinker around and see how the variables change. Also possible to use some other open data sets and plug them in.

Machine Learning will be a big part of the future, and already is, so learning it / having a look at it is definitely not a bad investment - and I find it to be a lot of fun :)

Go is not a traditional language for this (like Matlab, Python scikit, R), but it's often used in conjunction with those tools for data processing and scaling out the models create with these other languages. Whether Go will be used for more model traning / evaluation remains to be seen, but it's certainly possible, although the libraries are nowhere near the maturity of these other battle tested things.

#### Resources

* [goml](https://github.com/cdipaolo/goml)

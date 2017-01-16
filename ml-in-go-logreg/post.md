Machine Learning is getting and has been getting lots of attention during the last few years. I spent some time on ML topics when I wrote my thesis and also on- and off after graduating. I find the whole field of Machine Learning and its many possibilities quite fascinating.

However, getting from being interested and fascinated to actual working code, which solves some kind of problem, can be quite daunting. How much previous knowledge does one need to get started? Do you need to fully grasp all those formulas and models? How should I preprocess my data? It's hard to reliably answer those questions, because it really depends on what one wants to achieve and how much energy and time one wants to exert in the process.

In this post, I will try to just ignore most of these questions and instead show some code which aims to showcase how a very simple Machine Learning application could look like using Go. This post won't cover any kind of ground regarding the theoretical background of the Machine Learning technique which is used. If you are interested in learning more about Machine Learning (which I would definitely recommend), you'll find some resources I've found useful at the end of the post.

So, with all of that out of the way, it's time to get going. We will use **Logistic Regression** to create a very basic model which can help us with a **Classification Problem**.

Ok, so first of all, a **Classification Problem** is simply a problem in which we try to find out, based on existing observations (data), in which category a new observation will (most likely) fall. You can think for example, of a mole or birthmark where we want to find out whether it is malignant or benign. We could use images of moles we've seen in the past and where we know, whether they were malignant or benign (our observations), to train a model try to predict whether a new birthmark might be cancerous or not.

**Logistic Regression** is the algorithm we will use to create this model. [Here](https://en.wikipedia.org/wiki/Logistic_regression) is the wikipedia link and I'm sure there are many other very good resources on this widely used algorithm. Suffice to say, that it is an algorithm which is very well suited for classification problems. We will get into a few aspects of the algorithm later in the code example, because we need to provide some parameters for the algorithm to work properly, but for now it should be enough for us to know that this suits our problem and that the library we are going to use implements it correctly.

The goal here is not, as stated above, to fully understand how **Logistic Regression** works, but rather to get a feeling of which general steps are involved when trying to solve a problem using Machine Learning. We also won't deal with *Big Data* in any way, but rather work with a very small data set of about 100 entries, so there will be little value in interpreting the exact results we get for our little contrived example. But, as you can imagine, the code and techniques shown in this example will also work on other, larger data sets.

We will use Go for our example, but Go is by no means a go-to language for Machine Learning problems. In this area `R`, `Matlab`, `python` and a few others really shine. However, Go is often used in conjunction with these languages to pre- and post-process data as well as to scale these models, which is something Go excels at.

There are a few Machine Learning libraries available in Go, but they are of course nowhere near the maturity of e.g.: [scikit-learn](http://scikit-learn.org/). But for our simple purposes, they will more than do. For this example, I used [goml](https://github.com/cdipaolo/goml). I also took a look at [golearn](https://github.com/sjwhitworth/golearn/), which looks promising as well.

`goml`'s API is pretty straightforward and its [documentation](https://godoc.org/github.com/cdipaolo/goml) is OK as well. 

Alright, let's get started!

## Data

First of all, we need some data. In this example, we use a very simple, csv-based testset of ....

Explain data set

Of course, if we just test our model on the same data we trained it on, it will work pretty well, but that doesn't tell us how well it will perform in the real world. For this reason, we separate the data into a training set (70%) and a test set (30%). We will use the training set to create our model and then evaluate its performance using the test set.

Using `goml`, we can easily load and parse csv data like this:

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

In a real world scenario, we would of course further process the data to, for example, remove outliers or normalize the data, but in this simple case, the data is already normalized and we can just go on.

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
//TODO: ML learning resources

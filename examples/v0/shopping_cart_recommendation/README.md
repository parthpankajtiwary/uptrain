<h1 align="center">
  <a href="https://uptrain.ai">
    <img width="300" src="https://user-images.githubusercontent.com/108270398/214240695-4f958b76-c993-4ddd-8de6-8668f4d0da84.png" alt="uptrain">
  </a>
</h1>

<h1 style="text-align: center;">Monitoring Biases in a Recommender System</h1>



**Objective**: We want to monitor the prediction of a recommender system using the UpTrain framework. Specifically, we want to check how close the predictions of the model are to the ground truth and also check if the model recommendations suffer from any biases (such as the popularity bias).

**Dataset and ML model**: In this example, we train a recommender system to recommend items to users based on their previous shopping history. The dataset is a subset of the [Coveo data challenge dataset](https://github.com/coveooss/SIGIR-ecom-data-challenge) and the model to train embeddings is the [Word2Vec model](https://en.wikipedia.org/wiki/Word2vec). 


Note: Requires [Gensim](https://pypi.org/project/gensim/) to be installed. We ran the following code successfully with Gensim version 4.3.0.

### Step 1: Train the model

Each product has a unique stock-keeping unit (**sku**) that is used as a product identifier. We use [Word2Vec models from Gensim](https://radimrehurek.com/gensim/models/word2vec.html#gensim.models.word2vec.Word2Vec) to learn a embeddings corresponding to each sku based on shopping sessions of the user.


```python
x_train_sku = [[e['product_sku'] for e in s] for s in data['x_train']]
model = Word2Vec(sentences=x_train_sku, vector_size=48, epochs=15).wv
```

### Step 2: Define a custom monitor (cosine distance between embeddings of predicted and selected items)

Next, we define a custom metric where we want to monitor the cosine distance between embedding vectors of predicted and selected items. Specifically, we want to measure the cosine distance between the ground truth and first predicted item.


```python
def cosine_dist_init(self):
    self.cos_distances = []
    self.model = model

def cosine_distance_check(self, inputs, outputs, gts=None, extra_args={}):
    for output, gt in zip(outputs, gts):
        if (not output) or (not gt):
            continue
        y_preds = output[0]
        y_gt = gt[0]
        try:
            vector_test = self.model.get_vector(y_gt['product_sku'])
        except:
            vector_test = []
        vector_pred = self.model.get_vector(y_preds)
        if len(vector_pred)>0 and len(vector_test)>0:
            cos_dist = cosine(vector_pred, vector_test)
            self.cos_distances.append(cos_dist)
            self.log_handler.add_histogram('cosine_distance', self.cos_distances, self.dashboard_name)
```

### Step 3: Define another custom monitor (price difference between predicted and selected items)

Next, we also add a custom metric to measure the absolute log ratio between the ground truth and prediction item prices


```python
def price_homogeneity_init(self):
    self.price_diff = []
    self.product_data = data['catalog']
    self.price_sel_fn=lambda x: float(x['price_bucket']) if x['price_bucket'] else None
    
def price_homogeneity_check(self, inputs, outputs, gts=None, extra_args={}):
    for output, gt in zip(outputs, gts):
        if (not output) or (not gt):
            continue
        y_preds = output[0]
        y_gt = gt[0]
        prod_test = self.product_data[y_gt['product_sku']]
        prod_pred = self.product_data[y_preds]
        if self.price_sel_fn(prod_test) and self.price_sel_fn(prod_pred):
            test_item_price = self.price_sel_fn(prod_test)
            pred_item_price = self.price_sel_fn(prod_pred)
            abs_log_price_diff = np.abs(np.log10(pred_item_price/test_item_price))
            self.price_diff.append(abs_log_price_diff)
            self.log_handler.add_histogram('price_homogeneity', self.price_diff, self.dashboard_name)
```

### Step 4: Define the prediction pipeline


```python
x_test = data['x_test']
y_test = data['y_test']
inference_batch_size = 10

def model_predict(model, x_test_batch):
    """
    Implement the model prediction function. 
    
    :model: Word2Vec model learned from user shopping sessions
    :x_test_batch: list of lists, each list being the content of a cart
    
    :return: the predictions returned by the model are the top-K
    items suggested to complete the cart.

    """
    predictions = []
    for _x in x_test_batch:
        key_item = _x[0]['product_sku']
        nn_products = model.most_similar(key_item, topn=10) if key_item in model else None
        if nn_products:
            predictions.append([_[0] for _ in nn_products])
        else:
            predictions.append([])

    return predictions
```

### Step 5: Define UpTrain config and initialize the framework


```python
cfg = {
    # Define your metrics to identify data drifts
    "checks": [
        {
            'type': uptrain.Monitor.POPULARITY_BIAS,
            'algorithm': uptrain.BiasAlgo.POPULARITY_BIAS,
            'sessions': x_train_sku,   
        },
        {
            'type': uptrain.Monitor.CUSTOM_MONITOR,
            'initialize_func': cosine_dist_init,
            'check_func': cosine_distance_check,
            'need_gt': True,
            'dashboard_name': 'cosine_distance'
        },
        {
            'type': uptrain.Monitor.CUSTOM_MONITOR,
            'initialize_func': price_homogeneity_init,
            'check_func': price_homogeneity_check,
            'need_gt': True,
            'dashboard_name': 'price_homogeneity'
        }
    ], 
    "retraining_folder": 'uptrain_smart_data', 
    "logging_args": {"st_logging": True},
}

framework = uptrain.Framework(cfg)
```


### Step 6: Ship your model in production with UpTrain


```python
for i in range(int(len(x_test)/inference_batch_size)):
    # Define input in the format understood by the UpTrain framework
    inputs = {'data': {"feats": x_test[i*inference_batch_size:(i+1)*inference_batch_size]}}
    
    # Do model prediction
    preds = model_predict(model, inputs['data']['feats'])

    # Log input and output to framework
    ids = framework.log(inputs=inputs, outputs=preds)
    framework.log(identifiers=ids, gts=y_test[i*inference_batch_size:(i+1)*inference_batch_size])
```
    
### Monitoring the hit-rate of the model

By applying a concept drift check on the model in prediction, UpTrain automatically monitors the performance of the model. In this case, the performance is defined as the hit rate, that is, the proportion of items that was boought by the user was actually recommended by the model. We observe an average hit-rate of around 0.1. 

<img width="600" alt="hit_rate" src="https://user-images.githubusercontent.com/5287871/217215003-e121b499-0e69-4e98-9cd6-2b931cc62056.png">

### Histogram plot for items with popularity

From the UpTrain dashboard, we can find the histogram for popularity bias. We can see that most of the items that are recommended have low popularity. Our model does not look to be suffering from popularity bias.

![popularity_bias](https://user-images.githubusercontent.com/5287871/217118816-ce8a0267-138d-4218-ba4d-499d9e262c43.png)


### Histogram plot for cosine distance between ground truth and prediction

In the dashboard, we can measure the cosine distance between the embeddings of the recommended items and the items that were actually bought. A lot of them have zero cosine distance (implying that the recommendations were spot on). Also, we observe that the predictions are concentrated around the low cosine distance (< 0.4) space.

![cosine_distance](https://user-images.githubusercontent.com/5287871/217118837-a66d9315-3c97-42af-94cc-24876881cb42.png)


### Histogram plot for absolute log price ratio between prediction and selected items

Finally, we also added a custom monitor where we wanted to check whether our model is providing outrageous recommendations (e.g., recommending washing machines when the user wants to buy just a washing detergent). In the below plot, we observe that the price range of most of the recommended items is close to the price of the actually bought item.

![price_homogeneity](https://user-images.githubusercontent.com/5287871/217118853-45ceafce-2e4a-4b22-bea8-dc3581bf5f84.png)


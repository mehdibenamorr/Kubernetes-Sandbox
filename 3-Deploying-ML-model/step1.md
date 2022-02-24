In this step, we will download all needed files of a built image classifier for Penguin types that were saved in Our [minio instance](https://console.share.pads.fim.uni-passau.de).

The model was trained using this [jupyter_notebook](https://github.com/asyaf/fun_mini_projects/blob/master/penguin_classification/transfer_learning_penguin.ipynb). 

[Streamlit](https://streamlit.io/) is an open-source Python library that makes serving ML models as web apps as easy as possible without any required prior knowledge in web development.

This will not include any monitoring or logging of the model which are to be expected in a proper deployment to production.

To explain briefly how Streamlit works, let's go over the code bit by bit.

**Function that implements the upload functionality of the web app.**

<pre>
import io
from PIL import Image
import streamlit as st
import torch
from torchvision import transforms

MODEL_PATH = 'pytoch_model.pt'
LABELS_PATH = 'classes.txt'

def load_image():
    uploaded_file = st.file_uploader(label='Pick an image to test')
    if uploaded_file is not None:
        image_data = uploaded_file.getvalue()
        st.image(image_data)
        return Image.open(io.BytesIO(image_data))
    else:
        return None
</pre>

**Loading the model and the classes**

<pre>
def load_model(model_path):
    model = torch.load(model_path, map_location='cpu')
    model.eval()
    return model

def load_labels(labels_file):
    with open(labels_file, "r") as f:
        categories = [s.strip() for s in f.readlines()]
        return categories
</pre>

**The inference function that takes the input image and return ranked scores for all the classes**

<pre>
def predict(model, categories, image):
    preprocess = transforms.Compose([
        transforms.Resize(256),
        transforms.CenterCrop(224),
        transforms.ToTensor(),
        transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
    ])
    input_tensor = preprocess(image)
    input_batch = input_tensor.unsqueeze(0)

    with torch.no_grad():
        output = model(input_batch)

    probabilities = torch.nn.functional.softmax(output[0], dim=0)

    all_prob, all_catid = torch.topk(probabilities, len(categories))
    for i in range(all_prob.size(0)):
        st.write(categories[all_catid[i]], all_prob[i].item())
</pre>

**The main function of the app**

<pre>
def main():
    st.title('Custom model demo')
    model = load_model(MODEL_PATH)
    categories = load_labels(LABELS_PATH)
    image = load_image()
    result = st.button('Run on image')
    if result:
        st.write('Calculating results...')
        predict(model, categories, image)
</pre>

In the next step, we will deploy the this model in the cluster.
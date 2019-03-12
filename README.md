This little repository contains an example on how to use Jupyter
notebooks with py-veo. The example uses a little VE kernel calculating
the average of a vector of 1 million random numbers and measures the
performance with the help of py-veosinfo calc_metrics().


## Install prerequisites

```
sudo yum install -y python-devel python-virtualenv
```

## Create and configure a virtualenv containing jupyter 

```
virtualenv jupy
source jupy/bin/activate
pip install --upgrade pip
pip install jupyter
pip install py-veosinfo
pip install numpy
pip install py-veo
pip install psutil
```

### Create a password
```
python -c 'from notebook.auth import passwd ; password = passwd() ; print password'
#-> Enter password: 
#-> Verify password: 
#-> sha1:1b1452c8da2c:fc4572b136f1d4e0c2735e5b21a2d1593c5eb884
```


### Generate a config and edit password
```
jupyter notebook --generate-config
#-> Writing default config to: /home/focht/.jupyter/jupyter_notebook_config.py
```

### Modify c.NotebookApp.password in generated config, put in password hash
```
vim /home/focht/.jupyter/jupyter_notebook_config.py
```

### Run the jupyter notebook server
```
jupyter notebook
```

## Copy jupyter notebook and load it

Copy the file *py-veo-perf.ipynb* to th ejupy/ directory,
open the web browser pointing to localhost:8888,
authenticate with the password and open the notebook file.

Needless to say, the machine on which you run this must have a VE card
installed.

The notebook looks like [this](py-veo-perf.md).

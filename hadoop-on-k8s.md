# How to install Hadoop on your local Kubernetes cluster

Okey this is not the easiest way of running Hadoop on your local computer and probably you should instead just install it locally.

However if you really insist doing this here's how:

1) Install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/), [minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) and Docker if you don't already have it. I recommend using package-manager like Chocolatey. Minikube should install with VirtualBox as default driver which I recommend. When starting minikube we should increase its memory limit since our Hadoop node's pods need at least 2GB: `minikube --memory 4096 --cpus 2 start` (minikube's default is 1GB). NOTE: actually the Hadoop cluster by default uses about 10GB in memory limits and about 3GB running memory. From what I looked my k8s will overprovision to 300% of its capacity limits but use far less.
2) Install [helm](https://docs.helm.sh/using_helm/). Then run `helm init`.
3) Now you should have everything installed, let's spin up our Hadoop cluster:
```
  helm install \
    --set yarn.nodeManager.resources.limits.memory=4096Mi \
    --set yarn.nodeManager.replicas=1 \
    stable/hadoop
```
The default replica amount is 2 but there isn't enough resources in minikube to create two pods with 2GB memory each. If you want to allow k8s to use more of your PC's computing power you should increase the minikube's limits once more [link](https://github.com/kubernetes/minikube/issues/567). We are using the Helm chart from here https://github.com/helm/charts/tree/master/stable/hadoop.

4) Open up the k8s dashboard: `minikube dashboard`. You should see the Hadoop pods initializing. If everything went well they should all have 1/1 pods running. If you're running out of memory try adding more to minikube.
5) Now that we have our cluster up and ready you should copy your assets to the NodeManager. Here we're using [WordCount](https://hadoop.apache.org/docs/stable/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html) as example, you can copy my version of it from [here](https://github.com/TeemuKoivisto/wordcount/raw/master/target/WordCount-1.0-SNAPSHOT.jar).
```bash
#!/bin/bash
# We are grepping here the autogenerated name of our NodeManager pod
POD_NAME=$(kubectl get pods | grep yarn-nm | awk '{print $1}')
# This is basically the same as `docker cp`
# NOTE: here I am assuming you're in the same folder as the downloaded JAR-file, otherwise update the path accordingly
kubectl cp WordCount-1.0-SNAPSHOT.jar "${POD_NAME}":/home

# SSH into the container, same as `docker exec`
kubectl exec -it "${POD_NAME}" bash
cd /home
mkdir input
echo Hello World Bye World > input/file01
echo Hello Hadoop Goodbye Hadoop > input/file02
/usr/local/hadoop/bin/hadoop fs -put input / # Put the data into the HDFS drive
/usr/local/hadoop/bin/hadoop jar WordCount-1.0-SNAPSHOT.jar com.mycompany.wordcount.WordCount /input /output
```
If everything went fine running that script should start logging execution information from Hadoop. Smart way to do this instead of copying would be to mount the data as a separate volume from your local machine. I did manage to create a local volume from my local directory but haven't figured out yet how to add it to the Hadoop's Helm chart.

6) When you installed Hadoop using Helm it outputted some useful commands, let's use port-forwarding to see the Yarn UI in our localhost:  
`kubectl get pods | grep yarn-rm | awk '{print $1}' | xargs -i kubectl port-forward -n default {} 8088:8088`  
You should now see it in http://localhost:8088.
7) If all went well running `/usr/local/hadoop/bin/hadoop fs -cat /output/part-r-00000` inside the NodeManager should produce:
```
Bye     2
Hadoop  2
Hello   2
World   2
```
And now you have it! But the word counts are wrong. Whops.

Hmm I'm not really how useful this setup is but well it was great exercise to me at least. Now I know not to use Kubernetes if I just can avoid it : ). Because to be honest configuring all this was _really_ complicated for such simple or well seemingly simple thing. Bit too many rough edges. But I'm hopeful it will easier in time.# How to install Hadoop on your local Kubernetes cluster

Okey this is not the easiest way of running Hadoop on your local computer and probably you should instead just install it locally.

However if you really insist doing this here's how:

1) Install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/), [minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) and Docker if you don't already have it. I recommend using package-manager like Chocolatey. Minikube should install with VirtualBox as default driver which I recommend. When starting minikube we should increase its memory limit since our Hadoop node's pods need at least 2GB: `minikube --memory 4096 --cpus 2 start` (minikube's default is 1GB). NOTE: actually the Hadoop cluster by default uses about 10GB in memory limits and about 3GB running memory. From what I looked my k8s will overprovision to 300% of its capacity limits but use far less.
2) Install [helm](https://docs.helm.sh/using_helm/). Then run `helm init`.
3) Now you should have everything installed, let's spin up our Hadoop cluster:
```
  helm install \
    --set yarn.nodeManager.resources.limits.memory=4096Mi \
    --set yarn.nodeManager.replicas=1 \
    stable/hadoop
```
The default replica amount is 2 but there isn't enough resources in minikube to create two pods with 2GB memory each. If you want to allow k8s to use more of your PC's computing power you should increase the minikube's limits once more [link](https://github.com/kubernetes/minikube/issues/567). We are using the Helm chart from here https://github.com/helm/charts/tree/master/stable/hadoop.

4) Open up the k8s dashboard: `minikube dashboard`. You should see the Hadoop pods initializing. If everything went well they should all have 1/1 pods running. If you're running out of memory try adding more to minikube.
5) Now that we have our cluster up and ready you should copy your assets to the NodeManager. Here we're using [WordCount](https://hadoop.apache.org/docs/stable/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html) as example, you can copy my version of it from [here](https://github.com/TeemuKoivisto/wordcount/raw/master/target/WordCount-1.0-SNAPSHOT.jar).
```bash
#!/bin/bash
# We are grepping here the autogenerated name of our NodeManager pod
POD_NAME=$(kubectl get pods | grep yarn-nm | awk '{print $1}')
# This is basically the same as `docker cp`
# NOTE: here I am assuming you're in the same folder as the downloaded JAR-file, otherwise update the path accordingly
kubectl cp WordCount-1.0-SNAPSHOT.jar "${POD_NAME}":/home

# SSH into the container, same as `docker exec`
kubectl exec -it "${POD_NAME}" bash
cd /home
mkdir input
echo Hello World Bye World > input/file01
echo Hello Hadoop Goodbye Hadoop > input/file02
/usr/local/hadoop/bin/hadoop fs -put input / # Put the data into the HDFS drive
/usr/local/hadoop/bin/hadoop jar WordCount-1.0-SNAPSHOT.jar com.mycompany.wordcount.WordCount /input /output
```
If everything went fine running that script should start logging execution information from Hadoop. Smart way to do this instead of copying would be to mount the data as a separate volume from your local machine. I did manage to create a local volume from my local directory but haven't figured out yet how to add it to the Hadoop's Helm chart.

6) When you installed Hadoop using Helm it outputted some useful commands, let's use port-forwarding to see the Yarn UI in our localhost:  
`kubectl get pods | grep yarn-rm | awk '{print $1}' | xargs -i kubectl port-forward -n default {} 8088:8088`  
You should now see it in http://localhost:8088.
7) If all went well running `/usr/local/hadoop/bin/hadoop fs -cat /output/part-r-00000` inside the NodeManager should produce:
```
Bye     2
Hadoop  2
Hello   2
World   2

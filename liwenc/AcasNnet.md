# 解析神经网络

[返回index](./index.md)

## AcasNnet 格式：

```c++
//被封装在AcasNeuralNetwork 类中。 然后 AcasNeuralNetwork 被封装在AcasParser类中。
class AcasNnet {
public:
    int symmetric;     //1 if network is symmetric, 0 otherwise
    int numLayers;     //Number of layers in the network
    int inputSize;     //Number of inputs to the network
    int outputSize;    //Number of outputs to the network
    int maxLayerSize;  //Maximum size dimension of a layer in the network
    int *layerSizes;   //Array of the dimensions of the layers in the network

    double *mins;      //Minimum value of inputs
    double *maxes;     //Maximum value of inputs
    double *means;     //Array of the means used to scale the inputs and outputs
    double *ranges;    //Array of the ranges used to scale the inputs and outputs
    double ****matrix; //4D jagged array that stores the weights and biases
                       //the neural network.
    double *inputs;    //Scratch array for inputs to the different layers
    double *temp;      //Scratch array for outputs of different layers
};
```
---

这里是调用`AcasParser`的构造函数，目的是初始化`AcasParser`的私有成员`AcasNeuralNetwork`，但其实最终执行的是`AcasNeuralNetwork`中的构造函数，目的是初始化`AcasNeuralNetwork`的私有成员`_network`，其中调用了AcasNnet类的`load_network(filePath: string)`
```cpp
AcasNeuralNetwork::AcasNeuralNetwork( const String &path )
    : _network( NULL )
{
    _network = load_network( path.ascii() );
}
```
```cpp
AcasNnet *load_network(const char* filename)
{
    //Load file and check if it exists
    FILE *fstream = fopen(filename,"r");

    //Initialize variables
    int bufferSize = 10240;
    char *buffer = new char[bufferSize];
    char *record, *line;
    int i=0, layer=0, row=0, j=0, param=0;
    AcasNnet *nnet = new AcasNnet();

    /**
     * 1.char *fgets(char *str, int n, FILE *stream)
     * 从指定的流stream读取一行，并把它存储在str所指向的字符串内。
     * 当读取(n-1)个字符时，或者读取到换行符时，或者到达文件末尾时，它会停止
     * 
     * 2.char *strstr(const char *haystack, const char *needle)
     * 该函数返回在haystack中第一次出现needle字符串的位置，如果未找到则返回null。
     */
    //Read int parameters of neural network
    line=fgets(buffer,bufferSize,fstream);
    while (strstr(line, "//")!=NULL)
        line=fgets(buffer,bufferSize,fstream); //skip header lines
    
    /**
     * strtok 类比 split("a;b;c;d", ";")
     */
    record = strtok(line,",\n");
    /**
     * NNet文件注释以//开头
     * 1. 第一行前四个值如下所示
     */
    nnet->numLayers    = atoi(record);
    nnet->inputSize    = atoi(strtok(NULL,",\n"));
    nnet->outputSize   = atoi(strtok(NULL,",\n"));
    nnet->maxLayerSize = atoi(strtok(NULL,",\n"));

    //Allocate space for and read values of the array members of the network
    /**
     * 一直没搞懂numLayers和layerSizes的区别，似乎后者才是真正的层数，包括输入输出层
     * layerSizes: Array<in>;
     * 给layerSizes数组赋值
     */
    nnet->layerSizes = new int[(nnet->numLayers)+1];
    line = fgets(buffer,bufferSize,fstream);
    record = strtok(line,",\n");
    for (i = 0; i<((nnet->numLayers)+1); i++)
    {
        nnet->layerSizes[i] = atoi(record);
        record = strtok(NULL,",\n");
    }

    //Load the symmetric paramter
    // 默认是0
    line = fgets(buffer,bufferSize,fstream);
    record = strtok(line,",\n");
    nnet->symmetric = atoi(record);

    //Load Min and Max values of inputs
    nnet->mins = new double[(nnet->inputSize)];
    line = fgets(buffer,bufferSize,fstream);
    record = strtok(line,",\n");
    for (i = 0; i<(nnet->inputSize); i++)
    {
        nnet->mins[i] = atof(record);
        record = strtok(NULL,",\n");
    }

    nnet->maxes = new double[(nnet->inputSize)];
    line = fgets(buffer,bufferSize,fstream);
    record = strtok(line,",\n");
    for (i = 0; i<(nnet->inputSize); i++)
    {
        nnet->maxes[i] = atof(record);
        record = strtok(NULL,",\n");
    }

    //Load Mean and Range of inputs
    // Mean values of inputs and one value for all outputs (used for normalization)
    nnet->means = new double[(((nnet->inputSize)+1))];
    line = fgets(buffer,bufferSize,fstream);
    record = strtok(line,",\n");
    for (i = 0; i<((nnet->inputSize)+1); i++)
    {
        nnet->means[i] = atof(record);
        record = strtok(NULL,",\n");
    }

    nnet->ranges = new double[(((nnet->inputSize)+1))];
    line = fgets(buffer,bufferSize,fstream);
    record = strtok(line,",\n");
    for (i = 0; i<((nnet->inputSize)+1); i++)
    {
        nnet->ranges[i] = atof(record);
        record = strtok(NULL,",\n");
    }

    //Allocate space for matrix of Neural Network
    //
    //The first dimension will be the layer number
    //The second dimension will be 0 for weights, 1 for biases
    //The third dimension will be the number of neurons in that layer
    //The fourth dimension will be the number of inputs to that layer
    //
    //Note that the bias array will have only number per neuron, so
    //    its fourth dimension will always be one
    //

    /**
     * 看到这里终于明白了为什么，一个加上input、output layers一共三层的神经网络，却在nnet中的numslayer只有2.
     * 这是因为nnet不是神经网络的文件，而是构建神经网络的文件，比如三层的神经网络5*50*5，对于构建它的权重矩阵，只需要2层
     * 需要一个50*5的w1[][],和一个5*50的w2[][]
     * w2*(w1*input) = output
     */

    /**
     * 对于一个5*50*5的神经网络，只有两个权重矩阵，因此第一重循环是<2的，相当于0-1
     * 对于第二维只有两个向量，分别是weights和bias，每一个又是一个二维向量，分别有size[layer+1]行和size[layer]列，对于matrix[0]，在此例中是50行5列
     * 对于每一行有size[layer]列，所以第二重循环中，从layer+1，改为了layer，为matrix[0][0][0]申请了一个5个double的空间
     * 但是对于bias只申请了1个大小的空间，这是符合逻辑的
     * 
     * 对于该例
     * martix[a][b][c][d]
     * a 取0,1 代表对于5*50*5的神经网络需要两个权重矩阵
     * b 取0,1 分别代表，0：weight矩阵，1：bias矩阵
     * c 矩阵行数，对于martix[0][0]，c=50，d=5；对于martix[1][0]，c=5，d=50
     */

    // 构造一个空的martix用于存放网络
    nnet->matrix = new double ***[((nnet->numLayers))];
    for (layer = 0; layer<(nnet->numLayers); layer++)
    {
        nnet->matrix[layer] = new double**[2];
        nnet->matrix[layer][0] = new double*[nnet->layerSizes[layer+1]];
        nnet->matrix[layer][1] = new double*[nnet->layerSizes[layer+1]];
        for (row = 0; row<nnet->layerSizes[layer+1]; row++)
        {
            nnet->matrix[layer][0][row] = new double[nnet->layerSizes[layer]];
            nnet->matrix[layer][1][row] = new double[1];
        }
    }

    //Iteration parameters

    layer = 0;
    param = 0;
    i=0;
    j=0;

    //Read in parameters and put them in the matrix
    // 填充martix
    while((line=fgets(buffer,bufferSize,fstream))!=NULL)
    {
        if(i>=nnet->layerSizes[layer+1])
        {
            if (param==0)
            {
                param = 1;
            }
            else
            {
                param = 0;
                layer++;
            }
            i=0;
            j=0;
        }
        record = strtok(line,",\n");
        while(record != NULL)
        {
            // 获取权重矩阵
            nnet->matrix[layer][param][i][j++] = atof(record);
            record = strtok(NULL,",\n");
        }
        j=0;
        i++;
    }
    nnet->inputs = new double[nnet->maxLayerSize];
    nnet->temp = new double[nnet->maxLayerSize];


    delete[] buffer;
    fclose(fstream);

    //return a pointer to the neural network
    return nnet;
}
```

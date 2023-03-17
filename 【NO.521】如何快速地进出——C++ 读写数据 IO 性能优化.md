# 【NO.521】如何快速地进出——C++ 读写数据 I/O 性能优化

I 即 Input 表示进，O 即 Output 表示出，任何的进出，如果你没有掌握正确的姿势，就容易让工具累趴，如上图所示。比如说，读写数据，如果没有掌握正确的 IO 姿势，你的计算机很容易就累了。

在多媒体技术高速发展的今天，数据的积累成指数级别的增长。数据的挖掘要求我们对海量的数据进行输入，进而处理与分析之后，将结果输出。很多时候，在算法处理数据耗时较短的情况的，数据的 I/O 成了减少时间、提高程序性能的瓶颈。

要改进 I/O 性能，一个最简单的方法就是使用不同的方法。下面着重比较 ifstream、ofstream、fread、fwrite、mmap 在读写方面的时间差异。

不谈数据处理只看是否到内存中的性能比较，就是耍流氓。譬如，ifstream 在读入的时候，天然就可以对数据进行处理，而 fread/mmap 本质上只是做了一个内存的映射，我们仍需进一步将映射入的数据进行结构化。

为了进行比较，我们假设构造了一个数据，有多行的数据，行内数据用逗号隔开。最后一列是二分类标签数据。数据格式示例如下所示（一行）

![img](https://pic4.zhimg.com/80/v2-b905e6d45aaf84d500e9fd6d25e277ef_720w.webp)

我们要做的，就是把这个数据读入到对应的结构中，之后再原封不动地写入到新文件中，进行计时。为了无误差的计时，计时我采用 chrono。

```text
auto s1 = chrono::steady_clock::now();
auto s2 = chrono::steady_clock::now();
some code
cout << "read data time: " << chrono::duration_cast<chrono::duration<double>>(s2 - s1).count() << " s" << endl;
```

## 1.ifstream/ofstream 读写

人狠话不多，直接上代码。直接读代码，写那么多汉字来解读代码，毫无意义，又不是小孩子了。读代码永远是学习编程最快的方式。

```text
#include<bits/stdc++.h>//万能头文件 
using namespace std;

struct Data {
	vector<double> features;
	int label;
	Data(vector<double> f, int l) : features(f), label(l) {
	}
};

string trainFile = "train_data.txt";
int featuresNum;
vector<Data> trainDataSet;

bool loadTrainData() {


	ifstream infile(trainFile.c_str());
	string line;

	if (!infile) {
		cout << "打开训练文件失败" << endl;
		exit(0);
	}

	while (infile) {
		getline(infile, line);
		if (line.size() > featuresNum) {
			stringstream sin(line);
			char ch;
			double dataV;
			int i;
			vector<double> feature;
			i = 0;

			while (sin) {
				char c = sin.peek();
				if (int(c) != -1) {
					sin >> dataV;
					feature.push_back(dataV);
					sin >> ch;
					i++;
				} else {
					cout << "训练文件数据格式不正确，出错行为" << (trainDataSet.size() + 1) << "行" << endl;
					return false;
				}
			}
			int ftf;
			ftf = (int)feature.back();
			feature.pop_back();
			trainDataSet.push_back(Data(feature, ftf));
		}
	}
	infile.close();
	return true;
}

string outputFile = "output_data.txt";
int storeData() {
	featuresNum = trainDataSet[0].features.size();
	string line;
	int i;
	ofstream fout(outputFile.c_str());
	if (!fout.is_open()) {
		cout << "打开输出文件失败" << endl;
	}
	for (i = 0; i < trainDataSet.size(); i++) {
		for (int j = 0; j < featuresNum; j++) {
			fout << trainDataSet[i].features[j] << ",";
		}
		fout << trainDataSet[i].label <<"\n";
	}
	fout.close();
	return 0;
}


int main(int argc, char* argv[]) {
	auto s1 = chrono::steady_clock::now();
	loadTrainData();
	auto s2 = chrono::steady_clock::now();
	cout << "time elapsed: " << chrono::duration_cast<chrono::duration<double>>(s2 - s1).count() << " s" << endl;
	storeData();
	auto s3 = chrono::steady_clock::now();
	cout << "time elapsed: " << chrono::duration_cast<chrono::duration<double>>(s3 - s2).count() << " s" << endl;

}
```

结果表明，在我这个机子上，50M 的文件，使用 ifstream 读和结构化数据用了 25 秒。ofstream 写数据用了 15 秒。好，下面，我们再来看一下，fread 和 fwrite 需要多少时间。

## 2.fread/fwrite(or sprintf) 读写

同时，我也使用 fread 和 fwrite 进行了测试。为了和 fwrite 的不可读结果做对比，同时我还测试了 sprintf。所用的代码如下。

```text
#include<bits/stdc++.h>//万能头文件 
#define FEATURE_NUM 1000
#define TRAIN_DATA_NUM 10000
using namespace std;

string trainFile = "train_data.txt";
int featuresNum;
const int MAXS = 8 * (FEATURE_NUM + 1)*TRAIN_DATA_NUM;
char buf[MAXS];
double features[TRAIN_DATA_NUM][FEATURE_NUM];
int labels[TRAIN_DATA_NUM];

void loadTrainData() {

	FILE *fp;
	fp = fopen(trainFile.c_str(), "rb");
	int len = fread(buf, 1, MAXS, fp);
	fclose(fp);

	buf[len] = '\0';
	double position = 1.0;
	int symbol = 0;
	int point = 0;
	int i, col, row = 0;
	double line[FEATURE_NUM + 2] = { 0 };
	i = 0;
	for (char *p = buf; *p && p - buf < len; p++) {
		if (row >= TRAIN_DATA_NUM) {
			break;
		}

		if (*p == ',') {
			if (symbol == 1)
				line[i] = -line[i];
			i++;
			point = 0;
			position = 1.0;
			symbol = 0;
			line[i] = 0.0;
		} else if (*p == '\n') {
			if (symbol == 1)
				line[i] = -line[i];
			int ftf = (int)line[FEATURE_NUM];
			for (col = 0; col < FEATURE_NUM; col++) {
				features[row][col] = line[col];
			}
			labels[row] = ftf;
			row = row + 1;
			i = 0;
			point = 0;
			position = 1.0;
			symbol = 0;
			line[i] = 0.0;
		} else if (*p == '.') {
			point = 1;
		} else if (*p == '-') {
			symbol = 1;
		} else {
			if (point == 0)
				line[i] = line[i] * 10 + *p - '0';
			else {
				position = position*0.1;
				line[i] += position*(*p - '0');
			}
		}

	}
}

/* 拷贝为编码格式*/ 
char outputSet[MAXS];
string output_data = "output_data.txt";
int storeData() {
	char *output = outputSet;
	char *output0 = output;
	featuresNum = 1000;	
	for (int i = 0; i < 8000; i++) {
		for (int j = 0; j < featuresNum; j++) {
			memcpy(output, &(features[i][j]), sizeof(double));
			output += sizeof(double);
			*output = ',';
			output++;
		}
		memcpy(output, &(labels[i]), sizeof(int));
		output += sizeof(int);
		*output = '\n';
		output++;
	}
	FILE *pf = fopen(output_data.c_str(), "w+");
	fwrite(output0, 1, output-output0, pf);
	fclose(pf);
	return 0;
}

int storeData1() {
	char *output = outputSet;
	char *output0 = output;
	int n;
	featuresNum = 1000;	
	for (int i = 0; i < 8000; i++) {
		for (int j = 0; j < featuresNum; j++) {
			n = sprintf(output, "%0.3f,",features[i][j]);
			output += n;
		}
		n = sprintf(output, "%d\n",labels[i]);
		output += n;
	}
	FILE *pf = fopen(output_data.c_str(), "w+");
	fwrite(output0, 1, output-output0, pf);
	fclose(pf);
	return 0;
}

int main(int argc, char* argv[]) {
	auto s1 = chrono::steady_clock::now();
	loadTrainData();
	auto s2 = chrono::steady_clock::now();
	cout << "time elapsed: " << chrono::duration_cast<chrono::duration<double>>(s2 - s1).count() << " s" << endl;
	storeData1();
	auto s3 = chrono::steady_clock::now();
	cout << "time elapsed: " << chrono::duration_cast<chrono::duration<double>>(s3 - s2).count() << " s" << endl;

}
```

fread 读入数据并进行处理，所用的时间为 0.3 s。fwrite 写入数据时间为 0.2 秒。sprintf 写入数据时间为 12 s。

## 3.mmap 读写

mmap 需要用到如下的头文件。

```text
#include <sys/stat.h>
#include <sys/types.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>
```

mmap 和 fread（fwrite）本质上几乎不耗时的，耗时的是字符串的处理。但是，如果不加字符串处理，拿内存映射的时间和 fstream 函数比，显然对于 fstream 是不公平的。

```text
#include<bits/stdc++.h>

//mmap 需要用到如下头文件
#include <sys/stat.h>
#include <sys/types.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>

#define FEATURE_NUM 1000
#define TRAIN_DATA_NUM 10000
using namespace std;

string trainFile = "train_data.txt";
int featuresNum;
const int MAXS = 8 * (FEATURE_NUM + 1)*TRAIN_DATA_NUM;
char buf[MAXS];
double features[TRAIN_DATA_NUM][FEATURE_NUM];
int labels[TRAIN_DATA_NUM];

void loadTrainData() {

	int fd = open("train_data.txt", O_RDONLY);
	const int data_sz = lseek(fd, 0, SEEK_END);
	char* buf = (char*)mmap(
	                NULL, data_sz, PROT_READ, MAP_PRIVATE, fd, 0);

	//FILE *fp;
	//fp = fopen(trainFile.c_str(), "rb");
	//int  = fread(buf, 1, MAXS, fp);
	//fclose(fp);

	int len = data_sz;

	//buf[len] = '\0';
	double position = 1.0;
	int symbol = 0;
	int point = 0;
	int i, col, row = 0;
	double line[FEATURE_NUM + 2] = { 0 };
	i = 0;
	for (char *p = buf; *p && p - buf < len; p++) {
		if (row >= TRAIN_DATA_NUM) {
			break;
		}

		if (*p == ',') {
			if (symbol == 1)
				line[i] = -line[i];
			i++;
			point = 0;
			position = 1.0;
			symbol = 0;
			line[i] = 0.0;
		} else if (*p == '\n') {
			if (symbol == 1)
				line[i] = -line[i];
			int ftf = (int)line[FEATURE_NUM];
			for (col = 0; col < FEATURE_NUM; col++) {
				features[row][col] = line[col];
			}
			labels[row] = ftf;
			row = row + 1;
			i = 0;
			point = 0;
			position = 1.0;
			symbol = 0;
			line[i] = 0.0;
		} else if (*p == '.') {
			point = 1;
		} else if (*p == '-') {
			symbol = 1;
		} else {
			if (point == 0)
				line[i] = line[i] * 10 + *p - '0';
			else {
				position = position*0.1;
				line[i] += position*(*p - '0');
			}
		}

	}
}

char outputSet[MAXS];
string output_data = "output_data.txt";
int storeData() {
	char *output = outputSet;
	char *output0 = output;
	featuresNum = 1000;
	for (int i = 0; i < 8000; i++) {
		for (int j = 0; j < featuresNum; j++) {
			memcpy(output, &(features[i][j]), sizeof(double));
			output += sizeof(double);
			*output = ',';
			output++;
		}
		memcpy(output, &(labels[i]), sizeof(int));
		output += sizeof(int);
		*output = '\n';
		output++;
	}
	int total = output-output0;
	//FILE *pf = fopen(output_data.c_str(), "w+");
	//fwrite(output0, 1, output-output0, pf);
	//fclose(pf);

	int fd = open(output_data.c_str(),O_RDWR|O_CREAT|O_TRUNC,0666);
	truncate(output_data.c_str(),total);
	void *dst_ptr=mmap(NULL,total,PROT_WRITE|PROT_READ,MAP_SHARED,fd,0);
	memcpy(dst_ptr,output0,total);
	return 0;
}

/*
int storeData1() {
	char *output = outputSet;
	char *output0 = output;
	int n;
	featuresNum = 1000;
	for (int i = 0; i < 8000; i++) {
		for (int j = 0; j < featuresNum; j++) {
			n = sprintf(output, "%0.3f,",features[i][j]);
			output += n;
		}
		n = sprintf(output, "%d\n",labels[i]);
		output += n;
	}
	FILE *pf = fopen(output_data.c_str(), "w+");
	fwrite(output0, 1, output-output0, pf);
	fclose(pf);
	return 0;
}
*/

int main(int argc, char* argv[]) {
	auto s1 = chrono::steady_clock::now();
	loadTrainData();
	auto s2 = chrono::steady_clock::now();
	cout << "time elapsed: " << chrono::duration_cast<chrono::duration<double>>(s2 - s1).count() << " s" << endl;
	storeData();
	auto s3 = chrono::steady_clock::now();
	cout << "time elapsed: " << chrono::duration_cast<chrono::duration<double>>(s3 - s2).count() << " s" << endl;
	cout << features[7999][999]<<endl;

}
```

mmap 只能在 linux 下使用，它读取并处理数据所用的时间为 0.15 s，写入数据到 txt 所用的时间为 0.1 s。

## 4.结论

综上所述，ifstream/ofstream 一秒钟可以处理 3 M的数据。

他们所用的时间大概是 fread/fwrite 的 100 倍，是 mmap 的 150 倍，这里多出来的 50 倍大概是我找的 linux 机器算得比较快。事实上，fread 和 mmap 差距不大，但是 mmap 只能在 linux 下使用，所以，一般没什么特殊要求，用 fread 就行了。

![img](https://pic1.zhimg.com/80/v2-493ce7676c4bb87291184633c944ba84_720w.webp)

原文地址：https://zhuanlan.zhihu.com/p/391123571

作者：linux
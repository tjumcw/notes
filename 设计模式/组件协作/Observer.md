## Observer

### 一、简介

#### 1、动机

- 软件构建过程中，需要为某些对象建立一种“通知依赖关系”
  - 一个对象（目标对象）的状态发生改变，所有的依赖对象（观察者对象）都能得到通知
- 如果这样的依赖关系过于紧密，软件就不能很好地抵御变化
- 使用面向对象技术，可以弱化这种依赖关系，并形成一种稳定的依赖关系，实现软件体系结构的松耦合。

#### 2、定义

- 定义对象间的一种一对多（变化）的依赖关系，以便当一个对象的状态发生变化时，所有依赖它的对象都得到通知并更新



### 二、示例背景

- 实现一个文件分割器（以前存储设备小，存东西可能需要分割成几个），需求如下：
  - 要有一个GUI界面，即实现MainForm类，包括一些展示的信息以及成员方法
  - 实现文件分割器类FilesPlitter，主要是实现split功能
  - 在MainForm的界面上要有文件分割的进度，即MainForm类要观察FilesPlitter进行split()的进度

#### 1、结构化方式

```c++
class MainForm : public Form
{
	TextBox* txtFilePath;		//Form类中的控件，细节不重要，一个用来选择文件路径的box
	TextBox* txtFileNumber;		//Form类中的控件，细节不重要，一个用来选择分割数量的box
	ProgressBar* progressBar;	//Form类中的控件，细节不重要，一个用来展示进度信息的bar

public:
	void Button1_Click(){		//点击事件

		string filePath = txtFilePath->getText();
		int number = atoi(txtFileNumber->getText().c_str());

		FileSplitter splitter(filePath, number, progressBar);	//创建一个分割器对象并初始化

		splitter.split();		//调用分割器的成员方法split()分割文件

	}
};

class FileSplitter
{
	string m_filePath;
	int m_fileNumber;
	ProgressBar* m_progressBar;	//Form类中的控件，细节不重要，一个用来展示进度信息的bar

public:
	FileSplitter(const string& filePath, int fileNumber, ProgressBar* progressBar) :
		m_filePath(filePath), 
		m_fileNumber(fileNumber),
		m_progressBar(progressBar){

	}

	void split(){

		//1.读取大文件

		//2.分批次向小文件中写入
		for (int i = 0; i < m_fileNumber; i++){
			//...
			float progressValue = m_fileNumber;
			progressValue = (i + 1) / progressValue;
			m_progressBar->setValue(progressValue);
		}

	}
};
```

- 违背了依赖倒置设计原则
  - 高层模块不应该依赖于低层模块，二者都应该依赖于抽象
  - 抽象不应该依赖实现细节，实现细节依赖于抽象
  - 依赖一般指编译时依赖，即A依赖B，即A编译时B需要存在才能编译通过
- FileSplitter中的ProgressBar产生了编译时依赖，是实现细节
  - 因为进度通知不一定会以这种进度条的形式展现，可能需求会发生变化，如数字显示百分比
  - 有可能不是GUI界面展示，是终端命令行打印*表示百分比，上面的只能给MainForm界面使用
- 理解这种观察模式，其实进度条就是表示通知，上面展现的需求即
  - MainForm类需要观察到文件分割进度变化
  - 不应该以具体的进度条定死了通知的形式，这种实现太detail了
  - 以抽象的形式表示通知，不应该以具体的控件表示通知



#### 2、Observer

- 针对上述的通知实现方法太过细节，用一个接口类表达抽象的通知机制

```c++
class IProgress{		//I表示接口interface，接口在C++中就表示抽象基类
public:
	virtual void DoProgress(float value)=0;		//value表示0~1之间的百分比
	virtual ~IProgress(){}
};
```

- 修改FileSplitter类，将具体的通知控件修改为抽象的通知机制

```c++
class FileSplitter
{
	string m_filePath;
	int m_fileNumber;
	//ProgressBar* m_progressBar;	//具体通知控件
	IProgress* m_iprogress;		//抽象通知机制

public:
    FileSplitter(const string& filePath, int fileNumber, IProgress* iprogress){
        //...
    }
    
    void split(){
        for(int i = 0; i < m_fileNumber; i++){
            //...
            
            if(!m_iprogress){
                float progressValue = m_fileNumber;
				progressValue = (i + 1) / progressValue;
                m_iprogress->DoProgress(progressValue)
            }
        }
    }
};
```

- 对应修改MainForm类
  - 不能直接传ProgressBar，进行解耦
  - 实现了IProgress的接口DoProgress，后续针对终端或者其他进展示形式只要
    - 通过扩展子类（地位等同MainForm的子类），如class MainTerminal:  public Terminal, public IProgress{};
    - 子类重写父类对应的虚函数DoProgress

```c++
class MainForm : public Form, public IProgress
{
	TextBox* txtFilePath;
	TextBox* txtFileNumber;

	ProgressBar* progressBar;

public:
	void Button1_Click(){

		string filePath = txtFilePath->getText();
		int number = atoi(txtFileNumber->getText().c_str());
        
		//构造函数第三个参数为IProgress*,this其实就相当于实现了IProgress虚基类（或者说接口）的子类对象的指针，满足多态父类指针指向子类对象
		FileSplitter splitter(filePath, number, this);   
		splitter.split();


	}

    //实现了IProgress类的接口，即子类MainForm复写虚函数，此处写不写virtual都行，考虑MainForm若再作为基类，加上也无妨
	virtual void DoProgress(float value){	
		progressBar->setValue(value);
	}
};
```

- FileSplitter类与任何界面类不存在耦合关系（虽然只写了MainForm，但不耦合任何其他界面类）
  - 不仅可以在windows使用，可用于其他场景
  - 通过IProgress的抽象接口实现，只依赖于抽象接口类IProgress

- 还可以进行一些小的优化

```c++
class FileSplitter
{
	string m_filePath;
	int m_fileNumber;
	//ProgressBar* m_progressBar;	//具体通知控件
	IProgress* m_iprogress;		//抽象通知机制

public:
    FileSplitter(const string& filePath, int fileNumber, IProgress* iprogress){
        //...
    }
    
    void split(){
        for(int i = 0; i < m_fileNumber; i++){
            //...
            
            {
                float progressValue = m_fileNumber;
				progressValue = (i + 1) / progressValue;
                onProgress(progressValue);
            }
        }
    }
protected:
    virtual void onProgress(float value){		//可以写为虚方法供子类重写
        if(!m_iprogress){
        	m_iprogress->DoProgress(value);		//更新进度条
        }
    }
};
```

#### 3、支持多个观察者

- 回到动机，一个依赖的改变，所有类都要观察到变化，继续优化增加容器存储观察者

```c++
//修改FileSplitter

class FileSplitter
{
	string m_filePath;
	int m_fileNumber;

	List<IProgress*>  m_iprogressList; // 
	
public:
    //需要修改构造函数，m_iprogressList不好直接通过构造函数构造，应该通过增加成员方法来add和remove观察者
	FileSplitter(const string& filePath, int fileNumber) :
		m_filePath(filePath), 
		m_fileNumber(fileNumber){

	}

	void split(){

		//1.读取大文件

		//2.分批次向小文件中写入
		for (int i = 0; i < m_fileNumber; i++){
			//...

			float progressValue = m_fileNumber;
			progressValue = (i + 1) / progressValue;
			onProgress(progressValue);					//发送通知
		}
	}

	void addIProgress(IProgress* iprogress){
		m_iprogressList.push_back(iprogress);
	}

	void removeIProgress(IProgress* iprogress){
		m_iprogressList.remove(iprogress);
	}

protected:
	virtual void onProgress(float value){		//遍历容器，对每个观察者都进行更新，就是调用各个不同观察者自己所重写的DoProgress方法
		
		List<IProgress*>::iterator itor=m_iprogressList.begin();

		while (itor != m_iprogressList.end() )
			(*itor)->DoProgress(value); 
			itor++;
		}
	}
};
```

- 这样MainForm中就可以通过创建FileSplitter来不断addIProgress和removeIProgress增加和减少观察者
  - 除了windows的窗口程序MainForm类外，还实现了ConsoleNotifier的观察者类
  - 还可以实现更多形式的观察者类，只要根据需求重写他们继承自接口类IProgress的虚函数DoProgress即可
  - FileSplitter遍历自己存放观察者的容器，对每个观察者的进度都进行更新，并根据多态机制调用各个观察者自己重写的DoProgress方法

```c++
class ConsoleNotifier : public IProgress {		//写的新的观察者
public:
	virtual void DoProgress(float value){		//重写DoProgress虚函数，即对应到实现需求中另外一种通知的形式
		cout << ".";
	}
};

class MainForm : public Form, public IProgress
{
	TextBox* txtFilePath;
	TextBox* txtFileNumber;

	ProgressBar* progressBar;

public:
	void Button1_Click(){

		string filePath = txtFilePath->getText();
		int number = atoi(txtFileNumber->getText().c_str());

		ConsoleNotifier cn;

		FileSplitter splitter(filePath, number);

		splitter.addIProgress(this); 		//订阅通知，那边有通知就会告知，即调用观察者重写的代码
		splitter.addIProgress(&cn);			//订阅通知，那边有通知就会告知，即调用观察者重写的代码

		splitter.split();

		splitter.removeIProgress(this);

	}

	virtual void DoProgress(float value){
		progressBar->setValue(value);
	}
};

```

- 对于上述设计，流程为：
  - 调用Button1_Click()时创建一个FileSplitter对象，增加两个观察者，分别为MainForm以及ConsoleNotifier类的对象
  - 调用splitter.split()，读取大文件后，分批次分别对每个文件进行写入操作后，调onProgress()方法
  - 对每个批次的小文件，都要在onProgress()方法中遍历观察者容器。对每个观察者调用对应重写接口类方法的DoProgress()，更新进度信息



### 三、要点总结

- 使用面向对象的抽象，Observer模式使得我们可以独立地改变目标与观察者，从而使二者之间的依赖关系达到松耦合
  - 随便增加多少个观察者，FileSplitter是稳定不变的
  - 可以支持任何观察者，支持任何平台开发使用文件分割器（结构化设计中，通知方式与windows的具体控件绑死）

- 目标发送通知时，无需指定观察者，通知（可以携带通知信息作为参数）会自动传播
- 观察者自己决定是否需要订阅通知，目标对象对此一无所知
<h1 align = "center">课程设计之金庸的江湖</h1>

<h5 align = "center">171860604 季义铮 2133441657@qq.com</h5>

<h5 align = "center">171860602 郭开天 870082743@qq.com</h5>

### 1.课程设计目标

+ 通过一个综合数据分析案例：”金庸的江湖——金庸武侠小说中的人物关系挖掘“，来学习和掌握MapReduce程序设计。通过本课程设计的学习，可以体会如何使用MapReduce完成一个综合性的数据挖掘任务，包括全流程的数据预处理、数据分析、数据后处理等。

### 2.任务描述

+ 通过金庸武侠小说中的人物关系挖掘，来学习和掌握MapReduce程序设计，熟悉和掌握以下MapReduce编程技能：
  + 在Hadoop中使用第三方的Jar包来辅助分析；
  + 掌握简单的MapReduce算法设计：
    + 单词同现算法；
    + 数据整理与归一化算法；
    + 数据排序；
  + 掌握带有迭代特性的MapReduce算法设计；
    + PageRank算法；
    + 标签传播（Label Propagation）算法。
+ 具体的任务要求分为以下六点
  +  数据预处理，从原始的金庸小说文本中，抽取出与人物互动相关的数据，而屏蔽掉与人物关系无关的文本内容，为后面的基于人物共现的分析做准备。
  + 人物同现统计，完成基于单词同现算法的人物同现统计。我们需要对人物之间的同现关系次数进行统计，同现关系次数越多，则说明两人的关系越密切。
  + 人物关系图构建与特征归一化，根据共现关系构建人物关系邻接表，并计算各邻接点的共现概率。
  + 基于人物关系图的PageRank计算，通过计算PageRank，我们就可以定量地金庸武侠江湖中的“主角”们是哪些。
  + 在人物关系图上的标签传播，为图上的顶点打标签，进行图顶点的聚类分析，从而在一张类似社交网络图中完成社区发现。
  +  分析结果整理

### 3.设计思路

+ 根据任务要求，任务总体分为四个步骤

  +  对小说的处理：数据预处理、同现统计、构建人物关系图、计算共现概率

    这部分主要任务是对段落进行分词，对分词后的结果进行单词同现算法，找出人物共现关系对并统计频次后，计算概率进行归一化处理，从而得到归一化的人物共现关系图的邻接表。

  +  PageRank算法

    这部分主要任务是基于人物共现关系图运行PageRank算法

  + 标签传播算法

    这部分主要任务是依据定义好的初始标签，使用标签传播算法对归一化的人物共现关系图进行处理，得到各个人物相应的标签，对得到的数据做可视化处理，使其特征能直观地显现出来。

  + 分析结果整理

    综合共现关系图邻接表、PageRank输出以及标签传播算法的结果，生成边和点集合，使用Gephi工具进行可视化操作。

### 4.实验分工

+ 季义铮：对小说的处理，集群运行调试，实验报告撰写
+ 郭开天：PageRank算法，标签传播算法，分析结果整理及可视化

### 5.实验步骤

#### 步骤一 处理小说

##### 1.1数据预处理

+ 本任务的主要工作是从原始的金庸小说文本中，抽取出与人物互动相关的数据，而屏蔽掉与人物关系无关的文本内容，为后面的基于人物共现的分析做准备。

```
输入：：1.全本的金庸武侠小说文集（未分词）；2. 金庸武侠小说人名列表。
输出：分词后，仅保留人名的金庸武侠小说全集。
```

+ 分词工具介绍

  **Ansj_seg**支持对中文文本进行分词，并且可以添加用户自定义的词典，这样它可以准确识别金庸武侠小说中的人名。实验中使用了用户自定义词典，以及用户自定义词典优先的分词，如果在字典里，就是人名。

+ 实现细节

  + **Mapper**

    + setup：按行读取上传到HDFS的人名列表文件（People_List_unique.txt），然后将其导入**Ansj_seg**工具的自定义的字典中，并归类为names。
    
    + map：使用**DicAnalysis**——用户自定义词典优先策略进行分词，对分词的结果逐个进行测试，如果词性为names，则在字典里，提取人名，写入key
    
  + **DRIVER**
    + 根据输入命命令设置输入文件目录，人名列表，输出文件目录。
  
+ **MapReduce 设计**

  + **Mapper**

  ```java
  public static class ProProcessingMapper extends Mapper<LongWritable, Text, Text, NullWritable> {
          @Override
          protected void setup(Context context) throws IOException, InterruptedException {
              String nameList = context.getConfiguration().get("namelist");
              FileSystem fileSystem = FileSystem.get(context.getConfiguration());
              FSDataInputStream inputstream = fileSystem.open(new Path(nameList));
              BufferedReader br = new BufferedReader(new InputStreamReader(inputstream));
              String nameline;
              while ((nameline = br.readLine()) != null) {
                  UserDefineLibrary.insertWord((new StringTokenizer(nameline)).nextToken(), "names", 1000);
              }
          }
  
          @Override
          protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
              String line = value.toString();
              Result result = DicAnalysis.parse(line);
              List<Term> terms = result.getTerms();
              StringBuilder sb = new StringBuilder();
              if (terms.size() > 0) {
                  for (int i = 0; i < terms.size(); i++) {
                      String word = terms.get(i).getName(); // 拿到词
                      String natureStr = terms.get(i).getNatureStr(); // 拿到词性
                      if (natureStr.equals("names")) {
                          sb.append(word + " ");
                      }
                  }
              }
              String res = sb.length() > 0 ? sb.toString().substring(0, sb.length() - 1) : "";
              context.write(new Text(res), NullWritable.get());
          }
      }
  
  ```

  + **Reducer**

  ```java
   public static class ProProcessingReducer extends Reducer<Text, NullWritable, Text, NullWritable> {
          @Override
          protected void reduce(Text key, Iterable<NullWritable> values, Context context)
                  throws IOException, InterruptedException {
              context.write(key, NullWritable.get());
          }
      }
  ```

##### 1.2特征抽取：人物同现统计

+ 在人物同现分析中，如果两个人在原文的同一段落中出现，则认为两个人发生了一次同现关系。我们需要对人物之间的同现关系次数进行统计，同现关系次数越多，则说明两人的关系越密切。

```
输入：任务1的输出；
输出：在金庸的所有武侠小说中，人物之间的同现次数。
```

+ 实现细节

  + **Mapper**
    + 在同一段中，人名可能多次出现，任务一只负责提取出所有的人名，没有剔除多余的人名，任务必须在输出同现次数之前处理冗余人名。在 Mapper 中创建一个集合，把所有人名放入集合中，集合会自动剔除冗余的人名。
    + 进行人物同现统计，遍历集合中的名字，如果两者名字不相同，遍历集合中的名字，如果两者名字不相同，则作为Map的输出，key设为<name1,name2>,value 设为1
  + **Reducer**
    + 同现次数统计：两个人物之间应该输出两个键值对，如“狄云”和“戚芳”，应该输出“狄云，戚芳  1”和“戚芳，狄云  1”。多个段落中允许输出相同的键值对，因此，Reducer 中需要整合具有相同键的输出，输出总的同现次数。

+ **MapReduce 设计**

  + **Mapper**

  ```java
  public static class CooccurrenceMapper extends Mapper<LongWritable, Text, Text, IntWritable> {
          @Override
          protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
              Set<String> set = new HashSet<>();
              String line = value.toString();
              String[] names = line.split(" ");
              set.addAll(Arrays.asList(names));
              for (String name : set) {
                  for (String other_name : set) {
                      if (name.equals(other_name)) {
                          continue;
                      } else {
                          context.write(new Text(name + "," + other_name), new IntWritable(1));
                      }
                  }
              }
          }
      }
  ```

  + **Reducer**

  ```java
     public static class CooccurrenceReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
          @Override
          protected void reduce(Text key, Iterable<IntWritable> values, Context context)
                  throws IOException, InterruptedException {
              int count = 0;
              for (IntWritable i : values) {
                  count += i.get();
              }
              context.write(key, new IntWritable(count));
          }
      }
  ```

##### 1.3特征处理：人物关系图构建与特征归一化

+ 当获取了人物之间的共现关系之后，我们就可以根据共现关系，生成人物之间的关系图了。为了使后面的方便分析，还需要对共现次数进行归一化处理：将共现次数转换为共现概率

```
输入：任务2的输出
输出：归一化权重后的人物关系图
```

+ 实现细节
  + **Mapper**
    + 确保人物的所有邻居输出到相同结点处理：在 Mapper 结点将输入的键值对“< 狄云，戚芳 > 1”拆分，输出新的键值对“< 狄云 > 戚芳 :1”，“狄云”的所有邻居会被分配给同一个 Reducer 结点处理。
  + **Reducer**
    + 在 Reducer 结点首先统计该人物与所有邻居同现的次数和 count，每个邻居的的同现次数除以 count 就得到共现概率。为了提高效率，在第一次遍历邻居的时候，可以把名字和共现次数保存在链表里，避免重复处理字符串。由于后续实验在**PageRank**计算中需要设置初始pr值0.1，为了算法性能的优化，直接在value字段前加上pr初始值0.1


+ **MapReduce** 设计

  + **Mapper**

  ```java
  public static class NormalizationMapper extends Mapper<LongWritable, Text, Text, Text> {
          @Override
          protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
              String line = value.toString();
              String[] kv = line.split(",");
              context.write(new Text(kv[0]), new Text(kv[1]));
          }
      }
  ```

  + **Reducer**

  ```java
  public static class NormalizationReducer extends Reducer<Text, Text, Text, NullWritable> {
          @Override
          protected void reduce(Text key, Iterable<Text> values, Context context)
                  throws IOException, InterruptedException {
              double count = 0;
              StringBuilder sb = new StringBuilder();
              List<String> list = new ArrayList<>();
              for (Text str : values) {
                  list.add(str.toString());//人名链表
                  String[] nv = str.toString().split("\\s+");
                  count += Integer.parseInt(nv[1]);
              }
  
              for (String text : list) {//遍历人名链表
                  String[] nv = text.split("\\s+");
                  double number = Integer.parseInt(nv[1]);
                  double scale = number / count;
                  sb.append(nv[0] + ":" + String.format("%.4f", scale) + ";");
              }
              // sb.insert(0, key.toString() + "\t" + "0.1#");
              sb.insert(0, key.toString() + "\t"+0.1+"[");
              String res = sb.toString().substring(0, sb.length() - 1)+"]";
  
              context.write(new Text(res), NullWritable.get());
          }
      }
  ```

#### 步骤二：基于人物关系图的 PageRank 计算

+ 经过数据预处理并获得任务的关系图之后，就可以对人物关系图作数据分析，其中一个典型的分析任务是：PageRank 值计算。通过计算 PageRank，我们就可以定量地获知金庸武侠江湖中的“主角”们是哪些。

```
输入：任务3的输出
输出：人物的PageRank值
```

+ PageRank原
  + PageRank 算法由 Google 的两位创始人佩奇和布林在研究网页排序问题时提出，其核心思想是：如果一个网页被很多其它网页链接到，说明这个网页很重要，它的 PageRank 值也会相应较高；如果一个 PageRank 值很高的网页链接到另外某个网页，那么那个网页的 PageRank 值也会相应地提高。
     相应地，PageRank 算法应用到人物关系图上可以这么理解：如果一个人物与多个人物存在关系连接，说明这个人物是重要的，其 PageRank 值响应也会较高；如果一个 PageRank 值很高的人物与另外一个人物之间有关系连接，那么那个人物的 PageRank 值也会相应地提高。一个人物的 PageRank 值越高，他就越可能是小说中的主角。
     PageRank 有两个比较常用的模型：简单模型和随机浏览模型。由于本次设计考虑的是人物关系而不是网页跳转，因此简单模型比较合适。简单模型的计算公式如下，其中 $B_i$ 为所有连接到人物 $i$的集合，$L_j$ 为人物$ j $对外连接边的总数

  $$
P{R_i} = \sum\limits_{(j,i) \in {B_i}} {\frac{{P{R_j}}}{{{L_j}}} \tag{1}}
  $$

  + 在本次设计的任务 3 中，已经对每个人物的边权值进行归一化处理，边的权值可以看做是对应连接的人物占总边数的比例。设$w_{j}(P_i)$表示人物$ i $在人物$ j $所有边中所占的权重，则 PageRank 计算公式可以改写为
  $$
  R({P_i}) = \sum\limits_{{P_j} \in {B_i}} {{w_{j}(P_i)}{*}R({P_j})} \tag{2}
  $$
  

  + PR值排名不变的比例随迭代次数变化的关系图如下，由于我们考虑的是找出小说中的主角，所以只要关心PR值前100名的人物的排名的变化情况，可以看到迭代次数在10以后，PR值排名不变的比例已经趋于稳定了，所以基于效率考虑，选取10作为PR的迭代次数。

    <img src='assets/lpa.jpg' align='center'>
  
    
  
+ 实现细节

  + **Mapper**
    + 创建邻接图和获取初始PR值0.1：逐行分析原始数据，以每个人的人名作为key值，读取归一化中设置的0.1，设为初始pr值
    + Map：产生两种类型的<key,value>：< 人物名，PageRrank 值 >，< 人物名，关系链表 >。
      + 第一个人物名是关系图中当前人物所连接到的人物名，其PR值由当前人物的PageRank值乘以出边在当前人物中的权重得到
      + 第二个人物名是本身人物名，为了保存连接信息，以保证完成迭代过程，添加"#"供区分
  + **Reducer**
    + Reduce：将同一人物名的<key,value>汇聚到一起，如果value是PR值，则累加到count变量，如果是关系链表则保存为namelist,输出键值对<key,(count#namelist)>,这样就完成了一次迭代过程
  + **Driver**
    + 迭代十次，每次迭代的输入是上一次迭代的结果，迭代的结果是下一次迭代的输入。

+ **MapReduce** 设计

  + **Mapper**

  ```java
  public static class PageRankMapper extends Mapper<LongWritable, Text, Text, Text> {
  
          @Override
          protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
              String line = value.toString();
              if (line.length() > 0) {
                  int index_t = line.indexOf("\t");
                  int index_l = line.indexOf("[");
                  int index_r = line.indexOf("]");
                  // int index_j = line.indexOf("#");
                  // double PR = Double.parseDouble(line.substring(index_t + 1, index_j));
                  double PR = Double.parseDouble(line.substring(index_t + 1, index_l));
                  String name = line.substring(0, index_t);
                  String names = line.substring(index_l + 1, index_r);
                  for (String name_value : names.split(";")) {
                      String[] nv = name_value.split(":");
                      double relation = Double.parseDouble(nv[1]);
                      double cal = PR * relation;
                      context.write(new Text(nv[0]), new Text(String.valueOf(cal)));
                  }
                  context.write(new Text(name), new Text("#" + line.substring(index_t + 1)));
              }
          }
      }
  ```
  
  + **Reducer**
  
  ```java
      public static class PageRankReducer extends Reducer<Text, Text, Text, Text> {
          @Override
          protected void reduce(Text key, Iterable<Text> values, Context context)
                  throws IOException, InterruptedException {
              String nameList = "";
              double count = 0;
              for (Text text : values) {
                  String t = text.toString();
                  if (t.charAt(0) == '#') {
                      nameList = t.substring(1);
                      //System.out.println("namelist:"+nameList);
                  } else {
                      count += Double.parseDouble(t);
                  }
              }
              context.write(key, new Text(String.valueOf(count) + nameList));
          }
      }
  ```
  
  + **Driver**

```java
 public static void main(String[] args) throws Exception {

        for (int i = 0; i < maxTime; i++) {
            Configuration conf = new Configuration();
            String[] otherArgs = new GenericOptionsParser(conf, args).getRemainingArgs();
            if (otherArgs.length != 3) {
                System.err.println("Usage: PageRank <in> <out>");
                System.exit(2);
            }
            Job job = Job.getInstance(conf, "PageRank");
            job.setJarByClass(PageRank.class);
            job.setMapperClass(PageRankMapper.class);
            job.setReducerClass(PageRankReducer.class);

            // job.setPartitionerClass();
            // job.setGroupingComparatorClass();

            job.setMapOutputKeyClass(Text.class);
            job.setMapOutputValueClass(Text.class);

            job.setOutputKeyClass(Text.class);
            job.setOutputValueClass(Text.class);

            if (i == 0) {
                FileInputFormat.addInputPath(job, new Path(otherArgs[1]));
                FileOutputFormat.setOutputPath(job, new Path(otherArgs[2] + i));
            } else {
                FileInputFormat.addInputPath(job, new Path(otherArgs[2] + (i - 1)));
                FileOutputFormat.setOutputPath(job, new Path(otherArgs[2] + i));
            }
            job.getConfiguration().set("times", String.valueOf(i));

            Path out = new Path(otherArgs[2] + i);
            FileSystem fileSystem = FileSystem.get(conf);
            if (fileSystem.exists(out)) {
                fileSystem.delete(out, true);
            }

            System.out.println("iteration " + i);

            if (!job.waitForCompletion(true)) {
                System.exit(1);
            }
        }
        System.exit(0);
    }
```

#### 步骤三：基于人物关系图的标签传播

+ 标签传播算法的基本思路是是通过已知节点的标签信息来预标记与其相邻或连接相对紧密的节点，每个待标记的节点都是根据其周围节点的标签选择与自己相关性最大的来决定自己的结果，最基础的标签传播算法是根据周围邻居的某种标签的数目多少来决定，总是选择数量最多的标签；而如果不同邻居的关系紧密程度不一致的话（通常体现在点的权重或边的权重），就形成了带权重的标签传播算法，与基础算法类似，这种算法统计其周围某种标签所占的权重和，选择权重最大的标签进行标记。

  在本次实验中由于第三步的输出结果就是不同人物之间的紧密程度，所以可以直接利用这一信息设计带边权重的标签传播算法初始化的信息可以根据十四部作品中的主角来决定，对于其他次要角色都初始化为0，他们的标签就是通过由每部作品主角所引伸出的关系网决定。考虑到大部分角色都是只出现在一部作品中，即使有部分出现在多部作品中的角色他们在不同作品中的关系网权重也是差别极大的，所以一旦认为我们的迭代结果覆盖了所有结点（这实际上只需要2-3次迭代就能达到），然后再进行一次迭代精确结果即可。最后的结果就是每个角色都有了一个标签，可以根据这一标签进行可视化实现。

+ 实现细节

  + **Mapper**
    + setup：初始化键值对列表，将原来初始化的标签信息读入内存中。
    + map：迭代计算，对每个节点都用一个数组储存周围所有节点的权重，分别求和后选择权重最大的非0标签设为自己新的标签。
    + cleanup:每次迭代后将新的结果储存到原来的文件中。
  + **Driver**
    + 设置输入，原始标签信息，输出。

+ **Mapreduce**设计

  + **Mapper**

  ```java
  public static class LabelPropagationMapper extends Mapper<Object, Text, Text, NullWritable> {
          HashMap<String, Integer> temp_label1 = new HashMap<String, Integer>();
          HashMap<String, Integer> temp_label2 = new HashMap<String, Integer>();
          FileSystem hdfs;
          String rawTag;
  
          // 用于初始化键值对列表
          public void setup(Context context) throws IOException {
              rawTag = context.getConfiguration().get("rawtag");
              hdfs = FileSystem.get(context.getConfiguration());
              Scanner tagFile = new Scanner(hdfs.open(new Path(rawTag)), "UTF-8");
              while (tagFile.hasNextLine()) {
                  StringTokenizer st0 = new StringTokenizer(tagFile.nextLine());
                  String word = st0.nextToken();
                  String tag = st0.nextToken();
                  temp_label1.put(word, Integer.parseInt(tag));
              }
              tagFile.close();
          }
  
          // 用于map集群计算
          public void map(Object key, Text value, Context context) throws IOException, InterruptedException {
              String line = value.toString();
              int index_t = line.indexOf("\t");
              int index_l = line.indexOf("[");
              int index_r = line.indexOf("]");
  
              String mainName = line.substring(0, index_t);
              String names = line.substring(index_l + 1, index_r);
              StringTokenizer st1 = new StringTokenizer(names, ";");
              if (!temp_label1.containsKey(mainName)) {
                  System.err.println(("error: " + mainName + " not found at " + context.getConfiguration().get("times")));
                  // System.exit(2);
                  return;
              }
              double[] my_list = new double[15];
              for (int i = 0; i < 15; i++)
                  my_list[i] = 0;
              while (st1.hasMoreTokens()) {
                  StringTokenizer st2 = new StringTokenizer(st1.nextToken(), ":");
                  String conName = st2.nextToken();
                  double conValue = Double.parseDouble(st2.nextToken());
                  if (!temp_label1.containsKey(conName))
                      continue;
                  int temp_value = temp_label1.get(conName);
                  my_list[temp_value] += conValue;
              }
              int temp_max = 1;
              for (int i = 2; i < 15; i++) {
                  if (my_list[i] > my_list[temp_max])
                      temp_max = i;
              }
              if (my_list[temp_max] > 0) {
                  temp_label2.put(mainName, temp_max);
                  context.write(new Text(mainName + String.valueOf(temp_max)), NullWritable.get());
              } else {
                  temp_label2.put(mainName, 0);
                  context.write(new Text(mainName + String.valueOf(0)), NullWritable.get());
              }
          }
  
          public void cleanup(Context context) throws IOException {
              hdfs.delete(new Path(rawTag), true);
              FSDataOutputStream out_put = hdfs.create(new Path(rawTag));
              PrintWriter pr1 = new PrintWriter(out_put);
              Set<Entry<String, Integer>> set = temp_label2.entrySet();
              Iterator<Entry<String, Integer>> iterator = set.iterator();
              while (iterator.hasNext()) {
                  Entry<String, Integer> entry = iterator.next();
                  pr1.println(new String(entry.getKey() + " " + entry.getValue()));
              }
              pr1.close();
              out_put.close();
          }
  
      }
  ```

#### 步骤四：分析结果整理及可视化

+ 可视化工具**Gephi** 是一款开源免费跨平台基于JVM的复杂网络分析软件,，其主要用于各种网络和复杂系统，动态和分层图的交互可视化与探测开源工具。
  + 该软件支持使用csv数据格式导入节点列表和边列表；同时使用节点属性进行渲染，数据格式如下：
    + 边列表：
    + 节点列表：
  + 节点有两种属性，LPA生成的标签和PageRank计算的PR值，标签决定节点显示的颜色，PR值决定标签的大小。权重决定节点与节点之间的距离。
+ 边列表和节点列表的生成通过**CsvGeneriter**类来实现
  + 实现细节
    + 通过PageRank计算结果设置标签的大小，标签传播的结果设置节点颜色，共现次数决定节点与节点之间的权重。

### 6.实验结果

##### 数据预处理

+ 执行方式

```
hadoop jar CurDsn.jar proprocessing  MP/Data/wuxia_novels  MP/Data/People_List_unique.txt MP/task1out
```

+ 文件位置:MP/task1out
+ 运行截图

<img src='assets/task1_运行.png' align='center'>

+ 结果展示

<img src='assets/task1.png' align='center'>

##### 同现统计

+ 执行方式

```
hadoop jar CurDsn.jar cooccurrence MP/task1out MP/task2out
```

+ 文件位置:/user/2020st30/output_2
+ 运行截图

<img src='assets/task2_运行.png' align='center'>

+ 结果展示

<img src='assets/task2out.png' align='center'>

##### 人物关系图构建及归一化

+ 执行方式

```
hadoop jar CurDsn.jar normalization MP/task2out MP/task3out
```

+ 文件位置:/user/2020st30/output_3
+ 运行截图

<img src='assets/task3_运行.png' align='center'>

+ 结果展示

<img src='assets/task3out.png' align='center'>

##### **PageRank**计算

+ 执行方式

```
hadoop jar Cur.jar pagerank MP/task3out Mp/task4out
```

+ 文件位置:Mp/taskout0 MP/task4out1…MP/task4out9

+ 运行截图

  + 第一次迭代

  <img src='assets/pr_0.png' align='center'>

  + 第十次迭代

  <img src='assets/pr_9.png' align='center'>

+ 结果展示

  + 第一次迭代

  <img src='assets/task4out0.png' align='center'>

  + 第十次迭代

  <img src='assets/task4out9.png' align='center'>

##### 标签传播

+ 执行方式

```
hadoop jar CurDsn.jar lpa   MP/task4out9 /user/2020st30/rawtag.txt MP/task5out
```

+ 文件位置:/MP/task5out2

+ 运行截图

  + 第一次迭代

  <img src='assets/lpa_0.png' align='center'>

  + 第二次迭代

  <img src='assets/lpa_1.png' align='center'>

  + 第三次迭代

  <img src='assets/lpa_2.png' align='center'>

+ 结果展示

  + 第一次迭代

  <img src='assets/task5out0.png' align='center'>

  + 第二次迭代

  <img src='assets/task5out1.png' align='center'>

  + 第三次迭代

  <img src='assets/task5out2.png' align='center'>

##### 可视化csv生成

+ 执行方式

```
java -jar CurDsn.jar csv pagerank relation tag
```

+ 结果展示
+ 结果分析

##### 可视化

+ 结果展示

![image-20200801094223552](assets\result)

<img src='assets/result_1.png' align='center'>

<img src='assets/result_2.png' align='center'>

<img src='assets/result_3.png' align='center'>

<img src='assets/result_4.png' align='center'>

+ 结果分析
+ 可以看到，图里囊括了金庸小说中出现的大部分人物，每本书里的人物聚集成一类，显示为相同的颜色。并且，角色越重要，其PR值越高，相应地其标签越大。书中的次要角色则以主角为中心分布。如《鹿鼎记》里的主角韦小宝，《倚天屠龙记》里的张无忌，主角贯穿了整本书，与书中的大部分人物关系都比较密切，这说明图像分析的结果是符合实际情况的。
  
+ 每部书里的人物关系图也可以反映人物之间的联系，例如《射雕英雄传》中黄蓉，郭靖，杨康，不同作品里的人物也会产生关联，例如黄蓉，郭靖与《神雕侠侣》中的杨过小龙女等也有很多联系。
  




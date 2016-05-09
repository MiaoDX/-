## 本科毕设论文 -- 六自由度相机精确重定位方法研究与嵌入式实现

分为多个文件，这样在使用 latex 模板时不必再分开了。

## 总体思路

###1.介绍
精确重定位的需求、历史、现状；使用场景

###2.理论方法介绍
Direct Relative Orientation
主要是 5 points
1.[Recent Developments on Direct Relative Orientation](http://www.vis.uky.edu/~stewe/publications/stewenius_engels_nister_5pt_isprs.pdf)
2.[An Efficient Solution to the Five-Point Relative Pose Problem](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.86.8769&rep=rep1&type=pdf)
3.[A 4-point Algorithm for Relative Pose Estimation of a Calibrated Camera with a Known Relative Rotation Angle](https://www.inf.ethz.ch/personal/pomarc/pubs/LiIROS13b.pdf)

###3.整个项目概览

·················
·               ·
·····           ·    <- 飞鹏的图
·   ·           ·
·················

左下角的 Gimbal 与 Platform 是我的毕设主要内容

提到：提供了精确的移动，便能由重定位算法指导实现精确重定位

###4.使用软件工程的一般方法来进行叙述
1.需求分析
2.概要设计
3.详细设计
4.编码实现
5.测试
6.软件交付、验收、维护

参考：
[需求分析，概要设计，详细设计的标准格式](http://blog.sina.com.cn/s/blog_49ed05e60100089i.html)
[需求分析、概要设计、详细设计析义（转）](http://blog.sina.com.cn/s/blog_5b7b30fb0100fkrp.html)
[需求分析、概要设计、详细设计等写法(仅供参考使用)](http://wenku.baidu.com/link?url=HeCmrzg6Rj9cAjoaGcwjdH_NTy6d0AqMpaPHRp-msRSfakIF8MINb4B09-RLscdvIv6QiuI6D29ndpBNOvBalXZoYPs3rf7YaG-a9hK_Sie)


###5.摘录自撰写规范
1.  绪论：作为论文的开端，简要说明作者所做工作的目的、范围、国内外进展情况、前人研究成果、本人的设想、研究方法等。
2.  正文：为毕业设计（论文）的核心部分，包括理论分析、数据资料、实验方法、结果、本人的论点和结论等内容，还要附有各种有关的图表、照片、公式等。要求理论正确、逻辑清楚、层次分明、文字流畅、数据真实可靠，公式推导和计算结果无误，图表绘制要少而精。
（1） 图：包括曲线图、示意图、流程图、框图等。图序号一律用阿拉伯数字分章依序编码，如：图1.3、图2.11。每一图应有简短确切的图名，连同图序号置于图的正下方。图中坐标上标注的符号和缩略词必须与正文中一致。
（2） 表：包括分类项目和数据，一般要求分类项目由左至右横排，数据从上到下竖列。分类项目横排中必须标明符号或单位，竖列的数据栏中不宜出现“同上” 、“同左”等类似词语，一律填写具体的数字或文字。表序号一律用阿拉伯数字分章依序编码，如：表2.5、表10.3。每一表应有简短确切的题名，连同表序号置于表的正上方。
（3） 公式：正文中的公式、算式、方程式等必须编排序号，序号一律用阿拉伯数字分章依序编码，如：式(3-32)、式(6-21)。对于较长的公式，另行居中横排，只可在符号处（如：+、-、*、/、< >等）转行。公式序号标注于该式所在行（当有续行时，应标注于最后 一行）的最右边。连续性的公式在“=”处排列整齐。大于999的整数或多于三位的小数，一律用半个阿拉伯数字符的小间隔分开；小于1的数应将0置于小数点之前。
（4）计量单位：单位名称和符号的书写方式一律采用国际通用符号。
3. 结论：是对主体的最终结论，应准确、完整、精炼。阐述作者创造性工作在本研究领域的地位和作用，对存在的问题和不足应给予客观的说明，也可提出进一步的设想。
4. 致谢：对协助完成论文研究工作的单位和个人表示感谢。
5. 参考文献：在学位论文中引用参考文献时，引出处右上角用方括号标注阿拉伯数字编排的序号（必须与参考文献一致）。参考文献的排列格式分为：
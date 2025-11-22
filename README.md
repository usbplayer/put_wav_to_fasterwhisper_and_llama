# put_wav_to_whisper_llama

多个流程放到转译和翻译文件夹

目前服务器上有多个流程，需要依次处理文件：

先进入环境： conda activate srt

在/mnt/fnos/音声/自己转换的/source_file/ 目录下，有多个文件夹，每个文件夹下面，分别有“音声”，“无SE”，和一个图片文件。

先以此判断，“音声” 文件夹内是否有音频文件，  和 图片文件是否存在 且唯一，两个条件，如果满足，继续执行。

然后对满足条件的文件依次执行：

先把整个文件夹移动到 /mnt/fnos/音声/自己转换的/doing/ 目录下，表示现在正在处理。

然后先执行  
docker stop llama_cpp-llama-full-1
docker stop faster-whisper-faster-whisper-rocm-pytorch-1
关闭两个容器作为初始化，当然如果已经关闭了就忽略

然后清空一下/mnt/disk1/dockerx/faster-whisper/source_file/ 文件夹，作为初始化

然后，检测 项目文件夹内的 “无SE” 文件夹 内，是否有音频，如果有，就把多个音频文件 复制到 /mnt/disk1/dockerx/faster-whisper/source_file/  文件夹内  ，不是文件夹，直接把文件放进去

等到所有文件复制完成，剪切到  /mnt/disk1/dockerx/faster-whisper/input/

记录一下移动进去的文件名称

然后

 docker restart faster-whisper-faster-whisper-rocm-pytorch-1

容器开启会自动处理 input 文件夹内的音频，处理完成的文件会移动到/mnt/disk1/dockerx/faster-whisper/output/

监控这个/mnt/disk1/dockerx/faster-whisper/output/路径，等所有记录的文件名称全部出现在output文件夹，加上 同名的srt文件，都出现在output文件夹后，说明该流程处理完毕

然后只保留srt文件，在 /mnt/fnos/音声/自己转换的/doing/ 的项目文件夹当中，新建一个文件夹叫“日文字幕”，把srt文件移动到 这个文件夹当中

清空 /mnt/disk1/dockerx/faster-whisper/下  input 、doing、 output 三个文件夹内所有内容。

执行  docker stop faster-whisper-faster-whisper-rocm-pytorch-1

接着先清空 /mnt/disk1/dockerx/llama_cpp/srt_translate/ 路径下面  input 、doing、 output  文件夹

然后把刚才移动到项目文件夹下的“日文字幕”当中的srt文件，复制到 /mnt/disk1/dockerx/llama_cpp/srt_translate/input/  

复制完成后，先运行  docker restart llama_cpp-llama-full-1  打开这个容器

然后 进入环境 conda activate srt ，执行 python /mnt/disk1/dockerx/llama_cpp/srt_translate/main.py

如此input当中文件会依次处理，直到output 当中出现 这些srt文件，以及他们文件名+_translated.srt的文件。

等所有文件都齐全了，在 /mnt/fnos/音声/自己转换的/doing/ 的项目文件夹当中，新建一个文件夹叫“中文字幕”，把srt文件移动到 这个文件夹当中

将  文件名+_translated.srt  这些汉化文件，移动到 中文字幕 文件夹 当中

代表这个项目文件夹所有文件已经处理完毕，可以去处理下一个项目文件夹了。

请以以上说明，写一个python脚本，要求是减少封装，不要过度包装。不要繁杂，能流程式处理就流程式处理，函数也尽量别用。除了我发的文字，不要其他注释。


-----

现在遇到一个问题，第二个/mnt/disk1/dockerx/llama_cpp/srt_translate/main.py 脚本执行的时候，可能会卡住。特征是：用watch rocm-smi 观察GPU%的占用情况，连续一分钟都是0%，遇到这种情况，就需要把这个脚本强退，然后把处理到一半的srt文件，从/mnt/disk1/dockerx/llama_cpp/srt_translate/doing拿回到 input，然后重新执行这个脚本，直至所有文件都出现在output。先单独输出一个函数，能一直打印GPU%的占用率，然后在另外一个线程当中，执行这个函数并做判断，如果以上情况出现，就开始杀掉脚本从头执行。不要从头生成脚本，说明从哪里开始改就行。

第二个情况，因为刚才中断，导致output下已经有三个文件了，是已经执行完毕的srt文件，我已经手动移动到项目原文档当中，所以当项目有“中文字幕”文件夹的时候，把名字去掉_translated ，  比对日文字幕文件夹的文件，看哪个是没有翻译的，然后下面这个步骤就只干没翻译的，已经翻译了跳过。这个逻辑也单独修改。

---

首先请回顾代码逻辑，请问如果项目文件夹内，存在“日文字幕”文件夹，是否会跳过整个第一个流程，因为最后出现大量的空文件夹。麻烦检查是否一开始就跳过了，或者流程有误，当出现什么错误的时候，已经转译好的srt文件没有移动到 日文字幕 文件夹而中断了。

其次是，中文字幕文件夹当中，出现了日文原字幕和带有_translate后缀的汉化字幕，后面流程当中本身会把input路径当中的文件移动到output当中，所以请识别其中这方面，把其中_translate 摘出来移动到汉化字幕，而不是全部。

还有，在整个脚本开始过程中，先扫描可以做的文件，其中先是doing当中的所有项目文件夹，其中可能有之前没做过的，或者没符合要求的，或者做了一半的，以及昨晚的，麻烦把音声文件夹当中文件名和日文字幕文件名做对比，看哪个音声文件没转录的，先记录下来后续发给转录的那个流程用，然后看看哪个是有日文字幕没有中文字幕的，记录下来，后续发给翻译流程用。最后，再才是扫描一开始的source_file路径，看哪个项目文档满足要求，然后开始执行流程。

最后，把所有流程里面的处理模式，由整个项目文件夹的所有文件一起移动，改为以具体文件为最小单位。首先源文件夹从在/mnt/fnos/音声/自己转换的/source_file 改为 /mnt/fnos/音声/自己转换的/todo，原因是我自己筛选一遍文件后放在todo。然后开始先处理遗留的doing，doing处理完毕后，再把开始处理的文件夹从todo移动到doing。先开始一个一个音声移动，转译出一个srt后，发给 日文字幕文件夹，然后就把这个srt翻译，翻译完成后，发给中文字幕文件夹。以此类推。如果doing路径下，项目文件夹的音声是空的，就把这个项目文件夹移动回source_file 。说明不具备处理条件。

改动较多，请从头生成脚本吧。麻烦只有体现不同流程的时候，才用函数封装。判断项目文件夹合不合要求的用一个函数，执行项目文件夹的移动的代码用一个函数，转译用一个函数，翻译用一个函数。只准再函数开头用三个引号写函数注释，其他任何地方都不要写注释。
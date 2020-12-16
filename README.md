# data_management
___
## 目录

* [一、项目背景](#一-项目背景)

* [二、流程](#二-流程)
	* [1、图片提取](#1-图片提取)
	* [2、标注](#2-标注)
	* [3、质检](#3-质检)
	* [4、生成数据](#4-生成数据)
	* [5、上传数据库](#5-上传数据库)
	
* [三、具体实施](#三-具体实施)
	* [1、图片提取](#1-图片提取)
	* [2、质检](#2-质检)
	* [3、数据生成](#3-数据生成)
	* [4、数据库设计](#4-数据库设计)
	* [5、数据上传](#5-数据上传)


## 一、项目背景

- 对现场拍摄视频进行目标检测。

## 二、流程
### 1.图片提取

首先提取所拍摄视频中的图片，可以设定每秒提取多少帧。

### 2.标注

将提取的图片交给标注公司标注，标注包括图片中物体的类别和矩形框位置，按照标注文档进行标注。

### 3.质检

对标注公司返回的标注结果进行质检，截取标注的矩形框并按类别分在各个类别的文件夹下，将所有类别图片统一放在*class_img_dir*文件夹下。

### 4.生成数据

生成图片信息的csv、txt文件等。

### 5.上传数据库

将标注矩形框等数据信息上传数据库。


## 三、具体实施
### 1.图片提取

- 按帧提取视频图片
```
def gen_images(video_path, save_dir):
    vc = cv2.VideoCapture(video_path)
    c = 0

    if vc.isOpened():
        rval, frame = vc.read()
    else:
        rval = False
    while rval:
        rval, frame = vc.read()
        # 每秒提取5帧图片
        if c % FRAME_FRE == 0:
            image_name = video_path.split("/")[-1].split(".mp4")[0] +("_")+ str('%06d' % c) + '.jpg'
            try:
                cv2.imwrite(save_dir + "/" + image_name, frame)
            except:
                continue
        c += 1
```

### 2.质检

- 生成各个类别的文件夹，并把相应图片放在文件夹下。
```
def gen_class_img(img_dir,class_img_dir):
    """
    generate folders for each category, with images for each category in each folder
    :param img_dir: the path for saving images
    :param class_img_dir: the total folder for all categories
    :return:
    """
    csv_df = ReadFile().read(type="label_csv")
    class_list = []

    for index, row in tqdm(csv_df.iterrows(),total=len(csv_df),ncols=80):
        class_name = row["class_name"]
        class_file = os.path.join(class_img_dir, "{}".format(class_name))
        if class_name not in class_list:
            class_list.append(class_name)
            #generate category folders
			mkdir.mkdir(class_file)     
        xmin, ymin, xmax, ymax = row["xmin"], row["ymin"], row["xmax"], row["ymax"]
        filename = row["filename"]
        image = cv_imread(os.path.join(img_dir, filename))
        cropimg = image[int(ymin):int(ymax), int(xmin):int(xmax)]
        img_path = os.path.join(class_file,filename.split('.jpg')[0] + '_' + xmin + '_' + ymin + '_' + xmax + '_' + ymax + '.jpg')
        cv2.imwrite(img_path, cropimg)
```

### 3.数据生成

- 生成类别信息的txt和json文件，生成图片信息的txt和csv文件。
#### （1）生成类别信息文件。
```
def gen_class_index_json_txt(data_dir, class_index_json_path, class_index_txt_path):
    """
    generate class_name and class_index json file.
    key: class name, value: class index
    @param data_dir: the folder is to save images and annotations
    @param class_index_json_path: the path for saving json file
    @param class_index_txt_path: the path for saving txt file
    @return:
    """

    class_index_dict = dict()

    xml_df = parse_xml.xml_parse(data_dir)
    class_name_list = set(xml_df["class"].values)

    for index, class_name in enumerate(class_name_list):
        class_index_dict[class_name] = index

    # save json file
    with open(class_index_json_path, "w") as json_file:
        json_file.write(json.dumps(class_index_dict))
    print("write {} finished !".format(class_index_json_path))
    print("num of classes is: {}".format(len(class_name_list)))

    # save txt file
    with open(class_index_txt_path, "w") as txt_file:
        index_class_dict = {value: key for key, value in class_index_dict.items()}
        for i in range(len(index_class_dict)):
            txt_file.write(index_class_dict[i])
            txt_file.write("\n")

    print("write {} finished !".format(class_index_txt_path))
```
#### （2）生成图片信息文件。
```
def xml_to_txt(data_dir, txt_dir, txt_name, class_index_json):
    """
    arrange label file from xml files, which is formatted as txt. For Example:

    image_full_path [space] [x_min, y_min, x_max, y_max, class_index [space]],

    @param data_dir: the folder is to save images and annotations
    @param txt_dir: the path for saving annotations
    @param class_index_json: this param is dictionary and key is class name, value is class index
    @return:
    """
    label_content_list = []
    class_dict = ReadFile.read_json_label(class_index_json)

    for xml_file in glob.glob(data_dir + '/*.xml'):
        tree = ET.parse(xml_file)
        root = tree.getroot()
        filename = root.find("filename").text
        abs_filename = os.path.join(data_dir, filename)
        total_obj_list = []
        for member in root.findall("object"):
            class_name = member[0].text
            x_min, y_min, x_max, y_max = int(member[4][0].text), int(member[4][1].text), int(member[4][2].text), int(member[4][3].text)
            #if class_name not in class_dict:
            #    error_str = "class_name:{} , xml_file:{}".format(class_name, xml_file)
            #    print(error_str)
            single_obj_str = ",".join([str(x_min), str(y_min), str(x_max), str(y_max), str(class_dict[class_name])])
            total_obj_list.append(single_obj_str)
        label_content_str = "{};{}".format(abs_filename, ";".join(total_obj_list))
        label_content_list.append(label_content_str)

    # write total label
    with open(os.path.join(txt_dir, txt_name), "w") as label_txt:
        for index, label_content in enumerate(label_content_list):
            label_txt.write(label_content)
            if index < len(label_content_list) - 1:
                label_txt.write("\n")
    print("write {} finished !".format(txt_name))
```

### 4.数据库设计
- 数据库连接
```
数据库连接方式为：
主机号：192.168.20.249，端口：3306
```
- 数据库的实施
```
数据库选取MySQL，数据库名称为：DB_Img，由img_basic、img_bbox、img_basic_origin、img_bbox_origin、spot_basic、class_common、special_info、class_common八个数据表组成。其中，img_basic_origin、img_bbox_origin为img_basic、img_bbox的备份表。如表所示。（若后续有其他新数据，可新建数据表）
```
表 数据库表的功能说明
|序号|表名|功能说明|
|----|----|----|
|1|img_basic|图片基本信息表，对图片信息添加、修改和查询|
|2|img_bbox|图片标注矩形框信息表，查询标注矩形框位置|
|3|spot_basic|地点信息表，描述图片对应地点|
|4|class_common|标注类别是否通用|
|5|special_info|场景中特殊情况描述|
|6|class_common|种类状态对应信息表|

### 5.数据上传

- 数据上传到数据库中，以便以后复用。
#### （1）上传类别信息、特殊信息、通用信息等。
```
class Insert:
    #连接数据库
    def connect_db(self):
        # connect to database
        mydb = mysql.connector.connect(
            host=self.host, user=self.user, passwd=self.passwd, database=self.dbname, buffered=True
        )
        mycursor = mydb.cursor()
        return(mydb,mycursor)

    #插入地点信息
    def insert_spot(self):
        mydb,mycursor = Insert.connect_db(self)

        while True:
            spot = input("请输入地点名称(输入End结束)：")
            if spot!="End":
                mycursor.execute("select spot_id from {} order by spot_id DESC limit 1".format("spot_basic"))
                final_id = mycursor.fetchall()
                if final_id != []:
                    insert_id = final_id[0][0] + 1
                else:
                    insert_id = 1

                mycursor.execute("insert into {} (spot_id,spot) values ({},'{}')".format("spot_basic",insert_id,spot))
                mydb.commit()
            else:
                break

        mycursor.close()
        mydb.close()
    
	#插入场景特殊信息
    def insert_special(self):
        mydb,mycursor = Insert.connect_db(self)

        while True:
            spot = input("请输入地点信息(输入End结束)：")
            if spot!="End":
                description = input("请输入特殊信息：")
                mycursor.execute("select special_id from {} order by special_id DESC limit 1".format("special_info"))
                final_id = mycursor.fetchall()
                if final_id != []:
                    insert_id = final_id[0][0] + 1
                else:
                    insert_id = 1

                mycursor.execute("insert into {} (special_id,spot,description) values ({},'{}','{}')".format("special_info",insert_id,spot,description))
                mydb.commit()
            else:
                break

        mycursor.close()
        mydb.close()
    
	#插入类别通用信息
    def insert_common(self):
        mydb,mycursor = Insert.connect_db(self)

        while True:
            state_name = input("请输入状态名称(输入End结束)：")
            if state_name!="End":
                is_common = input("请输入是否通用(通用为YES,非通用为NO)：")
                if is_common=="YES" or is_common=="NO":
                    mycursor.execute("insert into {} (state_name,is_common) values ('{}','{}')".format("state_common",state_name,is_common))
                    mydb.commit()
                else:
                    print("输入错误，请重新输入：")
            else:
                break

        mycursor.close()
        mydb.close()

    #插入类别和状态对应信息
    def insert_class_state(self):
        mydb,mycursor = Insert.connect_db(self)

        while True:
            class_name = input("请输入种类名称(输入End结束)：")
            if class_name!="End":
                state_name = input("请输入状态名称：")
                mycursor.execute("insert into {} (class_name,state_name) values ('{}','{}')".format("class_state",class_name,state_name))
                mydb.commit()
            else:
                break

        mycursor.close()
        mydb.close()
```
#### （2）上传图片信息。
```
def insert(img_basic_origin_sql,img_bbox_origin_sql, dbname, host, user, passwd, first_insert):
    #connect to database
    mydb = mysql.connector.connect(
        host=host, user=user, passwd=passwd, database=dbname,buffered = True
    )
    mycursor = mydb.cursor()
	
    # 设置索引insert_times_index，表示第几次插入数据
    null_sql = "select count(*) from {}".format("img_basic_origin")
    mycursor.execute(null_sql)
    rowcount = mycursor.fetchall()[0][0]
    insert_times_index = 1

    if rowcount!=0:
        final_sql = "select insert_times_index from {} order by id DESC limit 1".format("img_basic_origin")
        mycursor.execute(final_sql)
        final_index = mycursor.fetchall()[0][0]
        insert_times_index = final_index+1

    #第一次插入新场景数据，插入次数设为1
    if first_insert==1:
        insert_times_index = 1


    scene = input("请输入场景名称(AGV/RGV):")
    spot = input("请输入地点名称:")

    # scene = "RGV"
    # spot = "guangzhou_nongxin_bank"
    mycursor.execute("select spot_id from {} where spot='{}'".format("spot_basic",spot))
    spot_id = mycursor.fetchall()[0][0]

    #读取数据
    total_csv_df = ReadFile().read(type="label_csv")
    img_basic_list = []
    img_bbox_list = []
    for i in tqdm(range(len(total_csv_df))):
        id = i+1+rowcount
        filename = total_csv_df["filename"][i]
        width = int(total_csv_df["width"][i])
        height = int(total_csv_df["height"][i])
        xmin = int(total_csv_df["xmin"][i])
        ymin = int(total_csv_df["ymin"][i])
        xmax = int(total_csv_df["xmax"][i])
        ymax = int(total_csv_df["ymax"][i])
        class_name = total_csv_df["class_name"][i]
        mycursor.execute("select state_name from {} where class_name='{}'".format("class_state", class_name))
        state_name = mycursor.fetchall()[0][0]
        bbox_ratio = round((xmax-xmin)*(ymax-ymin)/(height*width),6)
        img_basic_origin_value = (id,filename,width,height,scene,spot_id,insert_times_index)
        img_basic_list.append(img_basic_origin_value)
        img_bbox_origin_value = (id, filename, xmin, ymin, xmax, ymax, class_name,state_name,bbox_ratio,insert_times_index)
        img_bbox_list.append(img_bbox_origin_value)

    #批量插入数据到img_basic_origin：(id,filename,path,width,height,depth,insert_times_index)
    mycursor.executemany(img_basic_origin_sql,img_basic_list)
    mydb.commit()
    print(mycursor.rowcount, "img_basic_origin记录插入成功！")

    # 批量插入数据到img_bbox_origin：(id,filename,xmin, ymin, xmax, ymax, class_id,insert_times_index)
    mycursor.executemany(img_bbox_origin_sql, img_bbox_list)
    mydb.commit()
    print(mycursor.rowcount, "img_bbox_origin记录插入成功！")


    mycursor.close()
    mydb.close()
```

## 四、配置文件
- 文件名称在*filename.yaml*文档中；
```
dir : "data/qingdao_ditie"
data_dir : "total_data"
index_dir : "class_index"
train_dir : "train_data"
val_dir : "val_data"
label_csv_dir : "label_csv"
label_txt_dir : "label_txt"
class_index_json : "class_index/ob_classes.json"
class_img_dir : "class_img_dir"
tfrecord : "tf_record"
pbtxt_path : "ob.pbtxt"
split_ratio : 0.05
```
- 数据库相关操作的基本信息在*utils_db.conf*文档中。
```
[create]
dbhost=192.168.20.249
dbport=3306
dbuser=root
dbpassword=sunwin
dbname=DB_Img
dbcharset=utf8
sql_path=/utils/utils_sql_create_DBImg

[insert]
img_basic_origin_sql=INSERT INTO img_basic_origin (id,filename,width,height,scene,spot_id,insert_times_index) VALUES (%s,%s,%s,%s,%s,%s,%s)
img_bbox_origin_sql=INSERT INTO img_bbox_origin (id,filename,xmin,ymin,xmax,ymax,class_name,state_name,bbox_ratio,insert_times_index) VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)
img_basic_sql=INSERT INTO img_basic (id,filename,width,height,scene,spot_id,insert_times_index) VALUES (%s,%s,%s,%s,%s,%s,%s)
img_bbox_sql=INSERT INTO img_bbox (id,filename,xmin,ymin,xmax,ymax,class_name,state_name,bbox_ratio,insert_times_index) VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)


[update]
csv_dir = qingdao_ditie/label_csv
csv_name = total_data.csv
```

---
title: Automation Process for Email Attachment Excel in Python
date: 2020-12-15 22:28:10
categories: [Data, BI]
tags: [python, automation, ETL, Excel]
toc: true
cover: /img/pythonexcelribbon.webp
thumbnail: /img/pythonexcelicon.png
---

Working with business data, **Excel spreadsheet** is the most common file type you might deal with in daily basis, because Excel is a dominated application in the business world. What is the most used way to transfer those Excel files for business operation team, obviously, it's **Outlook**, because email attachment is the easiest way for business team to share data and reports. Following this business common logic and convention, you may get quite a lot Excel files from email attachment when you involved into a business initiative or project to design a data solution for BI reporting and business analysis.

The pain points is too much manual work dragging down efficiency of data availability and also increasing the possibility of human error. Imagine, every day get data from email attachment, you need to check your inbox every once for a while, then download those files from attachment, open Excel to edit data or rename file to meet data process requirement such as remove the protected password, after those preparation works all done, push data to NAS drive, finally, launch the job to proceed the data. It's not surprise that how easily you might make mistake because any single step contains error would cause the whole process failed.

It's very necessary to automate the whole data process if business project turns to BAU (Business As Usual) program and you have to proceed data in regular ongoing basis. `Python` and `Windows Task Scheduler` provides a simple and fast way to solve this problem, now let's take a look.

Overall speaking, this task can be broken down by a couple of steps:

1. Access Outlook to read email and download the attachment
2. Remove Excel file protected password (it's common in business file to protect data privacy)
3. Manipulate and edit Excel file
4. Copy file to NAS drive
5. Setup the Windows Task Scheduler to run python script automatically.

![emailexcelmindset.png](/img/screenshots/emailexcelmindset.png)

<!-- more -->

## Interact Outlook and Excel with Python

In this case, we are focusing on local operation because server setting is vary for different production environment. Python standard library can't meet our need on this request, we have to leverage its third-party libraries to access outlook and interact with Excel so we are going to import `win32com`, `openpyxl` and `shutil`. We firstly work out workflow to guide us to compose the python scripts

![attachmentexceltaskworkflow.png](/img/screenshots/attachmentexceltaskworkflow.png)

From the workflow chart, we can tell there are four functions in the streamline. The first one is read email and download attachment `readmail()`, this step is critical and the most import business logic build-in it.

![readmail_fn.png](/img/screenshots/readmail_fn.png)

we are going to apply two business logics, one is the latest email, the other one is if the email is latest (current date) then check if the current date -1 equals to data date in the subject line. we will process file when all the requirements are satisfied.

For Excel manipulation and spreadsheet decryption are pretty straight forward, no additional business logic so just apply the single function.  We create 3 python module files then encapsulate them into the main readmail() function script.

### fileTransfer_module.py

```python
import win32com.client as win32
import os
import os.path
import openpyxl as xl
from datetime import datetime, timedelta
import sys
import shutil

def cpfile(src_file, tgt_file):
    if os.path.exists(tgt_file):
        os.remove(tag_file)
        shutil.copy(src_file, tgt_file)
        
	else:
        shutil.copy(src_file, tgt_file)
if __name__ == "__main__":
    cpfile(src_file, tgt_file)
```

### excelOp_mudule.py

```python
import openpyxl as xl
from file_Transfer_module import cpfile
def xltemp(file, file_ind):
    prd_dir = r"\\your NAS drive UNC address"
    if file_ind == 0: ## for dealing with multiple Excel files
        fn = "yourExcelFile_1.xlsx"
        desc_fn = prd_dir + os.sep + fn
        cpfile(file, desc_fn)
	elif file_ind == 2:
        fn = "yourExcelFile_2.xlsx"
        desc_fn = prd_dir + os.sep + fn
        cpfile(file, desc_fn)
	elif file_ind == 1: ## Excel file need to be manipulated
        wb_raw = xl.load_workbook(file)
        fn_tgt = "yourExcelFile_3.xlsx"
        desc_file = prd_dir + os.sep + fn_tgt
        wb_tgt = xl.Workbook()
        ws_tgt = wb_tgt.active
        ws_tgt.title = "Sheet1" ## define work sheet name
        ws_tgt['A1'] = "Date" ## define the first column name
        ws_tgt['B1'] = "ID"
        ws_tgt['C1'] = "Type"
        ws_tgt['D1'] = "Amt"
        ws_tgt['E1'] = "Info"
        cur_sheet = str((datetime.today().day) - 1)
        ws_src = wb_raw.get_sheet_by_name(cur_sheet)
        1 = 1
        row_lt = []
        for each in ws_src.rows:
            if each[0].value is not None:
                row_lt.append(i)
                i += 1
            else:
                break
	    num_max_row_raw = row_lt.pop()
        num_max_col = ws_tgt.max_column
        for i in range(2, num_max_row_raw+1):
            for j in range(1, num_max_col+1):
                c = ws_src.cell(row = i, column = j)
                ws_tgt.cell(row = i, column = j).value = c.value
                
		wb_tgt.save(desc_file)
         wb_tgt.close()
         wb_raw.close()
if __name__ == "__main__":
    xltemp(file, file_ind)
```

### pwdDecry_module.py

```python
import win32com.client as win32
import os
import os.path
from datetime import datetime
import sys

def xlpwd():
    '''
    function xlpwd() is aiming to peel off attachment excel file password
	to be able to be ready by program
	arg: opt1_opt2
	value: "opt1" to deal with one excel file; "opt2" to deal with another excel file
    '''
	opt1_opt2 = sys.argv[1]
	excel = win32.Dispatch('Excel.Application')
	mon = '0' + str(datetime.today().month) ## suppose password is letters and 2 digits month combination
	if opt1_opt2 = "opt1":
    	pwd = "randomletters" + mon
    	fn = "yourExcelFile_1.xlsx"
    	sn = 2 ## sheet number
    	st = "tab name 1"
	elif opt1_opt2 = "opt2":
    	pwd = "otherrandomletters" + mon
    	fn = "yourExcelFile_2.xlsx"
    	sn = 1
    	st = "other tab name"
	prd_dir = r"\\your NAS UNC address"
	file = prd_dir + os.sep + fn
	wb_tgt = excel.Workbooks.open(file,0,False,5,pwd) ## open encrypted Excel file
	wb_tgt.Password = "" ## remove password
	wb_tgt.Worksheets(sn).Name = st ## define the tagart worksheet order and name
	wb_tgt.Save()
	wb_tgt.Close()
	excel.Quit()
if __name__ == "__main__":
    xlpwd()
```
### readMail.py

```python
import os, os.path
import sys
from datetime import datetime, timedelta
import win32com.client as win32
from excelOp_mudule import xltemp

def readMail():
    '''
    function readMail is aiming to read outlook email attachment and download them
    arg: lob, app, c_type
    value: lob == 'you line of business' and app == 'application name generate the source file' and c_type == 'business category'
    '''
    lob = sys.argv[1]
    app = sys.argv[2]
    c_type = sys.argv[3]
    outlook = win32.Dispatch("Outlook.Application").GetNameSpace("MAPI")
    type1 = outlook.Folders["your email address"].Folders["your inbox subfolder1"]
    type2 = outlook.Folders["your email address"].Folders["your inbox subfolder2"]
    type3 = outlook.Folders["your email address"].Folders["your inbox subfolder3"]
    ## if your inbox subfolder name has convention you can use while loop
    inbox = outlook.GetDefaultFolder(6) ## Microsoft Outlook API number for inbox is 6
    type1_msgs = type1.Items
    type2_msgs = type2.Items
    type3_msgs = type3.Items
    inbox_msgs = inbox.Items
    file_ind = 0
    folder = ""
    if lob == "finance" and app == "app1" and c_type == "consumer":
        mail_items, file_ind, folder = type1_msgs, 0, "finance_files"
	elif lob == "marketing" and app == "app2" and c_type == "small business":
        mail_items, file_ind, folder = type2_msgs, 1, "small business files"
	elif lob == "inventory" and app == "app3" and c_type == "cooperate":
        mail_items, file_ind, folder = type3_msgs, 2, "cooperate files"
	else:
        mail_items = None
	path = r"\\your file staing folder directory" + os.sep + folder
    num_mails = len(mail_items)
    lst_mails = list(reversed(range(num_mails)))
    id_mail = lst_mail[0]
    email = mail_items[id_mail]
    subject = email.Subject ## Outlook API Subject line object
    if file_ind in (0, 2):
        num = -13 ## depends on your own situation
        date_mail_str = subject[num:]
        if date_mail_str[0] != ' ':
            date_mail_dt = datetime.strptime(date_mail_str, "%b. %d, %Y")
		else:
            date_mail_dt = datetime.strptime(date_mail_str, " %b. %d, %Y")
	elif file_ind == 1 and datetime.today().strftime("%a") != "Mon" and datetime.today().day <= 10:
        date_mail_str = subject[-8:-1] + '2020'
        date_mail_dt = datetime.strptime(date_mail_str, " %B %d %Y")
	elif file_ind == 1 and datetime.today().strftime("%a") != "Mon" and datetime.today().day <= 10:
        date_mail_str = subject[-8:-1] + '2020'
        date_mail_dt = datetime.strptime(date_mail_str, "%B %d %Y")
	received_time = email.ReceivedTime ## Outlook API receive time object
    today = datetime.today()
    ## check availiability of the latest file
    if today.year == received_time.year and today.month == received_time.month and today.day == received_time.day:
        avail_ind = 1
	else:
        raise AttributeError("the latest file is not available! check with business team")
	## check if the file is right copy
    if avail_ind == 1 and (file_ind == 0 or file_ind == 1):
        val_dt = received_time - timedelta(days = 1)
        if date_mail_dt.day == val_dt.day and date_mail_dt.month == val_dt.month and date_mail_dt.year == val_dt.year:
            valid_copy_ind = 1 ## usually business file for current date is yesterday's data if data is daily basis
		else:
            raise AttributeError("file copy is not right! check with business team")
	elif avail_ind == 1 and file_ind == 2:
        val_dt = received_time
        if date_mail_dt.day == val_dt.day and date_mail_dt.month == val_dt.month and date_mail_dt.year == val_dt.year:
            valid_copy_ind = 1 ## sometime current date file is current date data depends on business process
		else:
            raise AttributeError("file copy is not right! check with business team")
	if valid_copy_ind == 1:
        attachment = email.Attachment.item(1)
        report_name = attachment.FileName
        os.chdir(path)
        input_file = os.getcwd() + os.sep + date_mail_str + report_name
        if not os.path.exists(input_file):
            attachment.SaveAsFile(input_file)
            xltemp(input_file, file_ind)

            
if __name__ == "__main__":
    readMail()
```

## Schedule jobs to auto check your inbox and execute python scripts

If you use Linux as your local machine then there are so many scheduling tools such as `crontab`, for Windows users, you can use GUI tool `Task Scheduler` to to the automation scheduling task. In this case, we use `Task Scheduler`. 

![taskscheduler.png](/img/screenshots/taskscheduler.png)

simply follow the wizard to create local jobs to implement above readMail.py and pwdDery_module.py scripts. After decrypted Excel files push to your server then trigger server jobs so that make the whole data process automated, no more manual work.










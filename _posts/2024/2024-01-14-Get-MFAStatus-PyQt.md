---
title: GUI_Get-MFAStatus
description: Python GUI frontend for the Get-MFAStatus.ps1 powershell script
author: BW
date: 2024-01-14 12:00:00 - 0500
categories: [Scripting, Python]
tags: [scripting]     # TAG names should always be lowercase
---

### GUI_Get-MFAStatus.pyw 

```python
#DESCRIPTION
#  Python GUI frontend for the Get-MFAStatus.ps1 powershell script
#  This script will get the Azure MFA Status of users. You can query all the users, admins only or a single user.
#  
#  It will return the MFA Status, MFA type and registered devices.

#NOTES
#  Name: GUI_Get-MFAStatus
#  Author: B. Werkman
#  Version: 1.1
#  DateCreated: Jan 2024

import sys
from PyQt5.QtWidgets import QApplication, QMainWindow, QTextEdit, QPushButton, QVBoxLayout, QHBoxLayout, QWidget, QInputDialog, QLineEdit, QLabel
from PyQt5.QtGui import QPixmap, QIcon
import subprocess
import os

class PowerShellGUI(QMainWindow):
    def __init__(self):
        super().__init__()
        self.initUI()
    def initUI(self):
        # Main window settings
        self.setWindowTitle('Get-MFAStatus')
        self.setGeometry(300, 300, 600, 400)
        self.image_label = QLabel(self)
         # Load an image with QPixmap and QIcon
        lib = "lib"
        image_dir = os.path.dirname(os.path.realpath(__file__))
        image_path = os.path.join(image_dir, lib, 'ITO.jpg')
        logo_path = os.path.join(image_dir, lib, 'logo.jpg')
        pixmap = QPixmap(image_path)  # Replace with your image path
        logomap = QIcon(logo_path)
        self.setWindowIcon(QIcon(logomap)) # Replace with your image path
         # Set the pixmap to the label
        self.image_label.setPixmap(pixmap)
        # Resize the label to fit the image
        self.image_label.resize(pixmap.width(), pixmap.height())
        # Text edit for output
        self.output_edit = QTextEdit(self)
        self.output_edit.setReadOnly(True)
        self.output_edit.setPlaceholderText("Output will be shown here...")
        # Buttons
        button_width = 120
        button_height = 50
        self.run_button = QPushButton('Get MFA Status', self)
        self.run_button.setFixedSize(button_width, button_height)
        self.run_button.clicked.connect(self.on_run)
        self.button2 = QPushButton('Query A User', self)
        self.button2.setFixedSize(button_width, button_height)
        self.button2.clicked.connect(self.on_button2_click)
        self.button3 = QPushButton('Without MFA Only', self)
        self.button3.setFixedSize(button_width, button_height)
        self.button3.clicked.connect(self.on_button3_click)
        self.button4 = QPushButton('Admins Only', self)
        self.button4.setFixedSize(button_width, button_height)
        self.button4.clicked.connect(self.on_button4_click)
        # Button layout
        button_layout = QHBoxLayout()
        button_layout.addWidget(self.run_button)
        button_layout.addWidget(self.button2)
        button_layout.addWidget(self.button3)
        button_layout.addWidget(self.button4)
        # Main layout
        layout = QVBoxLayout()
        layout.addLayout(button_layout)
        layout.addWidget(self.output_edit)
        layout.addWidget(self.image_label)
        # Central widget
        central_widget = QWidget()
        central_widget.setLayout(layout)
        self.setCentralWidget(central_widget)

    def append_with_formatting(self, line):
        cursor = self.output_edit.textCursor()

        # Example: Color lines containing 'failed' or 'error' in red
        if "failed" in line.lower():
            cursor.insertHtml('<span style="color: red;">' + line + '</span><br>')
        elif "finished" in line.lower():
            cursor.insertHtml('<span style="color: green;">' + line + '</span><br>')
        else:
            cursor.insertHtml('<span style="color: blue;">' + line + '</span><br>')

    def on_run(self):
        self.output_edit.setHtml('<span style="color: orange;font-weight: bold;">Opening Login prompt..Please wait..Do not close window until processing completes.</span>')
        QApplication.processEvents()
        script_dir = os.path.dirname(os.path.realpath(__file__))
        script_path = os.path.join(script_dir, 'Get-MFAStatus.ps1')
        # Construct the PowerShell command
        # Note: We separate the script path and switches as different arguments in the list
        command = ["powershell", "-ExecutionPolicy", "Bypass", "-File", script_path]
        # Run the command in background without displaying the terminal window
        creationflags = subprocess.CREATE_NO_WINDOW if os.name == 'nt' else 0
        # Run the PowerShell script with the switches
        process = subprocess.Popen(command, creationflags=creationflags, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
        output, error = process.communicate()
        # Show the output and error, if any
         # Process and append output line by line
        for line in output.decode('utf-8').split('\n'):
            self.append_with_formatting(line)

    def on_button2_click(self):
        text, okPressed = QInputDialog.getText(self, "Query A User", "Enter Username:", QLineEdit.Normal, "")
        if okPressed and text != '':
            self.output_edit.setHtml(f'<span style="color: green;">User entered: {text}</span>')
            self.output_edit.setHtml('<span style="color: orange;font-weight: bold;">Opening Login prompt..Please wait..Do not close window until processing completes.</span>')
            QApplication.processEvents()        
        else:
            self.output_edit.setHtml('<span style="color: orange;font-weight: bold;">Warning: No Username Entered</span>')
            QApplication.processEvents()
        script_dir = os.path.dirname(os.path.realpath(__file__))
        script_path = os.path.join(script_dir, 'Get-MFAStatus.ps1')
        switches = "-UserPrincipalName"
        command = ["powershell", "-ExecutionPolicy", "Bypass", "-File", script_path] + switches.split() + text.split()
        # Run the command in background without displaying the terminal window
        creationflags = subprocess.CREATE_NO_WINDOW if os.name == 'nt' else 0
        # Run the PowerShell script with the switches
        process = subprocess.Popen(command, creationflags=creationflags, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
        output, error = process.communicate()
        # Show the output and error, if any
        for line in output.decode('utf-8').split('\n'):
            self.append_with_formatting(line)

    def on_button3_click(self):
        self.output_edit.setHtml('<span style="color: orange;font-weight: bold;">Opening Login prompt..Please wait..Do not close window until processing completes.</span>')
        QApplication.processEvents()
        script_dir = os.path.dirname(os.path.realpath(__file__))
        script_path = os.path.join(script_dir, 'Get-MFAStatus.ps1')
        # Get the switches from the text input
        switches = "-withOutMFAOnly"
        # Construct the PowerShell command
        # Note: We separate the script path and switches as different arguments in the list
        command = ["powershell", "-ExecutionPolicy", "Bypass", "-File", script_path] + switches.split()
        # Run the command in background without displaying the terminal window
        creationflags = subprocess.CREATE_NO_WINDOW if os.name == 'nt' else 0
        # Run the PowerShell script with the switches
        process = subprocess.Popen(command, creationflags=creationflags, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
        output, error = process.communicate()
        # Show the output and error, if any
        for line in output.decode('utf-8').split('\n'):
            self.append_with_formatting(line)

    def on_button4_click(self):
        self.output_edit.setHtml('<span style="color: orange;font-weight: bold;">Opening Login prompt..Please wait..Do not close window until processing completes.</span>')
        QApplication.processEvents()
        script_dir = os.path.dirname(os.path.realpath(__file__))
        script_path = os.path.join(script_dir, 'Get-MFAStatus.ps1')
        # Get the switches from the text input
        switches = "-adminsOnly"
        # Construct the PowerShell command
        # Note: We separate the script path and switches as different arguments in the list
        command = ["powershell", "-ExecutionPolicy", "Bypass", "-File", script_path] + switches.split()
        # Run the command in background without displaying the terminal window
        creationflags = subprocess.CREATE_NO_WINDOW if os.name == 'nt' else 0
        # Run the PowerShell script with the switches
        process = subprocess.Popen(command, creationflags=creationflags, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
        output, error = process.communicate()
        # Show the output and error, if any
        for line in output.decode('utf-8').split('\n'):
            self.append_with_formatting(line)

def main():
    app = QApplication(sys.argv)
    ex = PowerShellGUI()
    ex.show()
    sys.exit(app.exec_())

if __name__ == '__main__':
    main()
```
{: file='GUI_Get-MFAStatus.pyw'}

import cv2
import time
import os , shutil
import numpy as np
from datetime import datetime
#from mail import sendMail 

fps = 10.1
res = '480p'

def getFileName():
    now = datetime.now()                                                              
    dt_string = now.strftime("%d_%m_%Y_%H:%M:%S")
    return dt_string + '.avi'
#Changes the resolution of the video
def changRes(vid,width,height):
    #4:3 ratio
    vid.set(3, width)
    vid.set(4, height)

standardDims = {
    "480p":(640,480),
    "720p":(1280,720),
    "1080p":(1920,1080),
    "4k":(3840,2160),
}

def getDimensions (vid, res='1080p'):
    width, height = standardDims["480p"]
    if res in standardDims:
        width,height = standardDims[res]
    #change the current caputre device to the resulting resolution
    changeRes(video, width, height)
    return width, height

VideoType = {
    'avi': cv2.VideoWriter_fourcc(*'XVID'),
    'mp4': cv2.VideoWriter_fourcc(*'XVID'),
}
#Returns Vieo file type
def getVidType(filename):
    filename, ext = os.path.splitext(filename)
    if ext in VideoType:
      return  VideoType[ext]

    return VideoType['avi']
    
firstFrame = None
motionDetected = False

#VideoCapture object created to record via web cam
video = cv2.VideoCapture(0)
out = cv2.VideoWriter(getFileName(), getVidType(getFileName()), fps, getDimensions(video,res))

fileName = getFileName()
seconds = 0

while True:
    seconds = seconds + 1/fps
    check, frame = video.read()
    #Converts the frame from color to grayscale
    gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
    #Converts the frame from grayscale to  GaussianBlur
    gray = cv2.GaussianBlur(gray,(21,21),0)

    #Stores the first frame of the video
    if firstFrame is None:
        firstFrame = gray
        continue

    #Calculates difference between first frame subsequent frames
    deltaFrame = cv2.absdiff(firstFrame,gray)
    #Provides a set threshold value, converting the difference value with less than 30 to black, otherwise it converts 
    #those pixels to white
    threshold = cv2.threshold(deltaFrame,30 ,255,cv2.THRESH_BINARY)[1]
    threshold = cv2.dilate(threshold, None, iterations = 0)
    #Add borders to outline
    (_,cnts,_) = cv2.findContours(threshold.copy(),cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
    #cnts = imutils.grab_contours(cnts)

    for contour in cnts:
        #Removes noises and shadows, only keeping areas with greater than 1000 pixels
        if cv2.contourArea(contour) < 1000:
            continue
        #Creats rectangular box around the object in the frame
        (x, y, w, h) = cv2.boundingRect(contour)
        print("motion detected")
        motionDetected = True
        cv2.rectangle(frame, (x,y), (x+w,y+h), (0,255,0), 3)
    #Videowriter saving video capture
    out.write(frame)
    cv2.imshow("frame",frame)
    cv2.imshow("delta",deltaFrame)
    cv2.imshow("thresh",threshold)
    
    #Every frame
    key = cv2.waitKey(1)
    print (seconds)
    #Records a five second clip
    if seconds >= 5:
        if motionDetected:
            #closes video file, saving to os, moving into videos folder
            out.release()
            shutil.move(fileName,'videos/'+ fileName)
            motionDetected = False
        else:
            #Closes videowriter object and immediately removes it
            out.release()
            os.remove(fileName)
        #reinitializes the videowriter object
        out.open(getFileName(), getVidType(getFileName()), fps, getDimensions(video,res))
        print (out.isOpened())
        fileName = getFileName()
        seconds = 0
    
    if key == ord('q'):
        break

video.release()
out.release()
os.remove(fileName)
cv2.destroyAllWindows





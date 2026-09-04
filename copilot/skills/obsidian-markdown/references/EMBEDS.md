# Embeds reference

~~~markdown
![Note Name](Note%20Name)
![](Note%20Name#Heading)
![](Note%20Name#^block-id)

![image.png](image.png)
![640x480](image.png)
![300](image.png)

![audio.mp3](audio.mp3)
![video.mp4](video.mp4)

![document.pdf](document.pdf)
![](document.pdf#page=3)
![](document.pdf#height=400)
~~~

A list embed needs a block ID after the list. An embedded search uses an
Obsidian query block:

~~~~markdown
~~~query
tag:#project status:done
~~~
~~~~

---
author: "Iestyn Green"
title: "TiL: Take full-length screenshots using DevTools"
description: "How to take full-length screenshots of webpages using DevTools"
draft: false
date: 2023-06-16
tags: ["TiL", "DevTools", "Screenshots", "Webpages", "Today I Learned"]
categories: ["TIL", "DevTools"]
ShowToc: true
TocOpen: true
---

**TL:DR Capture full-length webpage screenshots using built in Developer Tools.**

Sometimes when developing a webpage you may need to send screenshots back and forth between members of your squad or business.

When doing this it is important that you capture the full webpage in order to really display what you're working on. Which is why introducing simple devtools may be of use.

In order to do this you will need to open your browser of choice and navigate to the page you would like to capture.

---
## Google Chrome AND Microsoft Edge

Both Chrome and Edge use the same shortcut and developer tools. So follow the steps below if you are a Chrome or Edge user.

1. Once on the page you would like to capture you should firstly access Developer Tools by clicking `Cmd` + `Opt` + `I` *(on Mac)* or `Ctrl` + `Shift` + `I` *(on Windows)*

![Chrome-Step-1.png](images%2FChrome-Step-1.png)

2. Then, press `Cmd` + `shift` + `P` *(on Mac)* or `Ctrl` + `Shift` + `P` *(on Windows)*.

![Chrome-Step-2.png](images%2FChrome-Step-2.png)

3. In the search bar, immediately after the word Run >, type "screenshot".

![Chrome-Step-3.png](images%2FChrome-Step-3.png)  

4. Select Capture full size screenshot, and Chrome will automatically save a full-page screenshot to your Downloads folder as a png and it will generate the name based off the webpage's URL.

![Chrome-Step-4.png](images%2FChrome-Step-4.png)

## Firefox

With Firefox there are two methods to allow you take to take full size screenshots, one is with Developer Tools and the other is with a simple right click. We will go through both methods below and you can decide which you prefer.

1. Firstly navigate to the page you would like to capture, and then access Developer Tools by clicking `Cmd` + `Opt` + `I` *(on Mac)* or `Ctrl` + `Shift` + `I` *(on Windows)*
![Firefox-Step-1.png](images%2FFirefox-Step-1.png)

2. Next you will want to open the developer tools setting menu by clicking on the three dots on the top right of the dev tools menu
![Firefox-Step-2.png](images%2FFirefox-Step-2.png)

3. Once this is opened you will want to enable the "Take a screenshot of the entire page" option by clicking the tick box under "Available Toolbox Buttons"
![Firefox-Step-3.png](images%2FFirefox-Step-3.png)
![Firefox-Step-3.1.png](images%2FFirefox-Step-3.1.png)

4. Once that is enabled you will now have access to the *"Take a screenshot of the entire page"* (the small camera) button.
   ![Firefox-Step-4.png](images%2FFirefox-Step-4.png)

5. When the button is clicked it will capture a screenshot of the entire page and will save it as a png to your downloads folder using the date and time the screenshot was taken followed by '-fullpage' as the name.
    ![Firefox-Step-4.1.png](images%2FFirefox-Step-4.1.png)
 
### Second method   

The second method is a lot simpler and doesn't require you to access the developer tools.
**It also allows you to just copy the full page screenshot if you don't want to download it to your device.**

1. Once on the page you would like to capture, you can right click and select 'Take Screenshot'
![Firefox-2-Step-1.png](images%2FFirefox-2-Step-1.png)
2. Next the screen will go dark and you will be given the option to Drag on the page to select the region you would like to capture. But to capture the full screen you must click on the 'Save full page' button
![Firefox-2-Step-2.png](images%2FFirefox-2-Step-2.png)
3. After the buttons clicked it will display the screenshot and give you the option to either download or just copy the screenshot
![Firefox-2-Step-3.png](images%2FFirefox-2-Step-3.png)
4. If the you choose to download it then is will get saved to your downloads folder as a png using then date and time followed by the page's title as the screenshots name.

## Safari
For Safari it requires a few extra steps, such as enabling the developer console.

### How to enable Developer menu in safari
1. Firstly you will need to navigate to safari's settings. This can be located by clicking the safari tab and selecting settings. Or by simply clicking `Cmd` + `,`
![Safari-Step-1.png](images%2FSafari-Step-1.png)
 
2. Next you will need to navigate to the "Advanced" tab in safari's settings and then enable "Show Develop menu in menu bar". Once this is ticked you will now have access to the Developer menu in safari.
![Safari-Step-2.png](images%2FSafari-Step-2.png)

### How to capture a screenshot in safari
Once you have got the developer console set up then you should be ready to capture a full page screenshot.

1. Once on the page you would like to capture you should firstly access Developer Tools by clicking `Cmd` + `Opt` + `I`
![Safari-Step-3.png](images%2FSafari-Step-3.png)

2. `Control-click` or `right-click` while hovering over the **< html >** tag. You’ll get a flyout menu, within which you can select **Capture Screenshot**.
![Safari_Step-4.png](images%2FSafari-Step-4.png)

3. Once clicked you will then be offered the choice of where you would like to save the screenshot and what you would like to name it.
![Safari-Step-5.png](images%2FSafari-Step-5.png)


## Why would I want this?
It is important to communicate updates that you are producing to your webpages. Often when you go to take a screenshot using your device you can only capture what is available on your screen at the time which, *unless you have a very large monitor*, may not capture all the information on the page.

Also when creating pull requests it is nice to include a screenshot of the changes you have implemented so that the approvers can view them without needing to spin up the changes locally.

### TL:DR

- You can capture full page screenshots in **Chrome, Edge, Firefox** and **Safari** using dev tools.
- To access Developer Tools *(if enabled)* you can use the shortcut `Cmd` + `Opt` + `I` *(on Mac)* or `Ctrl` + `Shift` + `I` *(on Windows)*
- Chrome and Edge use the same screenshotting method, Firefox allows you to copy rather than download the screenshot if you choose and Safari allows you to name and select where the screenshot gets saved.
---

+++
title = "Command line tooling in Go"
description = "Tips, tooling recommendations for creating interactive CLIs in Go using tview, cobra, viper"
date = "2019-09-23"
tags = ["go", "command line tools"]
+++

I've been writing a lot of command line tooling the last few months; mainly at [work](https://www.bigcommerce.com), mainly in [Go](https://golang.org). Exposure to Go in various contexts has pulled me away from writing so many things in [Groovy](http://www.groovy-lang.org), particularly Groovy scripts.
Scripting in Groovy is nice as you can write fairly complex/involved apps using Grapes to
pull in artifacts via [Maven](https://maven.apache.org) with ease:

```groovy
#!/usr/bin/env groovy

@Grapes([
    @Grab(group='com.amazonaws', module='aws-java-sdk-rekognition', version='1.11.117'),
    @Grab(group='com.google.guava', module='guava', version='21.0')
])

import com.amazonaws.*

// does thing with guava and aws libs
```

With Go the workflow is a bit different, there's quite a bit more code, but the trade off
offers speed and a binary that doesn't require the JVM and the associated startup costs.
There are quite a number of good tools / libraries to be found on [awesome go](https://awesome-go.com).
Some of the listed tools I use, mainly:

- [viper](https://github.com/spf13/viper) for configuration
- [cobra](https://github.com/spf13/cobra) for command creation, documentation, etc...

Most tools do not present interfaces, however most is not all. For command line tools
that require or benefit from interactive widgets, flows, etc... I've found [tview](https://github.com/rivo/tview)
to be powerful and enormously useful. The tview project allows you to create modals,
forms with auto complete, inputs with validation, etc...

***

Take, for example, the [interactive-nomad-attach](https://github.com/joshdurbin/interactive-nomad-attach) project. This project interacts
with [Nomad](http://nomadproject.io) to get a list of jobs, allocations, task groups, and allows a user run a remote exec of an arbitrary command (in this case `/bin/bash`). This is identical to running `docker exec $id -it $command`, only from afar.

In this project, we leverage tview's:

- Modals
- Dropdowns
- Page functionality
- Queued updates
- Tview "app" suspension which is required to hand the full stdin/out/err to Nomad and the attached container

See the above link for the full source code. The following set of snippets contains references to functions in the full
version, found in the aforementioned link, or references have been straight up removed for simplicity.

1. Struct definition of widgets, pages, and the app:

  ```go
  type app struct {
  	*tview.Application
  	pages       *tview.Pages

  	// widgets need to be broken out individually so we can cross pull values while on other views or pages
  	welcomeModal             *tview.Modal
  	reattachModal            *tview.Modal
  	selectAllocationDropdown *tview.DropDown
  	selectJobDropdown        *tview.DropDown
  	selectTaskGroupDropdown  *tview.DropDown
  }
  ```

2. Instantiation of tview objects

```go
app := &app{
  Application:              tview.NewApplication(),
  pages:                    tview.NewPages(),
  welcomeModal:             tview.NewModal().SetText(welcomeModalText),
  selectAllocationDropdown: tview.NewDropDown().SetLabel(selectAllocationDropdownLabel),
  selectJobDropdown:        tview.NewDropDown().SetLabel(selectJobDropdownLabel),
  selectTaskGroupDropdown:  tview.NewDropDown().SetLabel(selectTaskGroupDropdownLabel),

  // don't set the text for the reattach modal as it's set via a queued updated in appContainerExec
  reattachModal: tview.NewModal(),
}
```  

3. Further definition of widgets, where pages are defined, and root objects are set within each page

```go
// add widgets to page object, only the first, the welcome modal, is initially set to visible
app.pages.AddPage(welcomeModalPageName, app.welcomeModal, true, true).
	AddPage(selectAllocationDropdownPageName, app.selectAllocationDropdown, true, false).
	AddPage(selectJobDropdownPageName, app.selectJobDropdown, true, false).
	AddPage(selectTaskGroupPageName, app.selectTaskGroupDropdown, true, false).
	AddPage(reattachModalPageName, app.reattachModal, true, false)

// define welcome modal
app.welcomeModal.AddButtons([]string{yesButton, noButton}).
	SetDoneFunc(func(buttonIndex int, buttonLabel string) {
		switch buttonLabel {
		case yesButton:
			app.pages.SwitchToPage(selectAllocationDropdownPageName)
		case noButton:
			app.pages.SwitchToPage(selectJobDropdownPageName)
		}
	})

// define allocation selection, once an option is selected execute container attachment
app.selectAllocationDropdown.
	SetOptions(app.getAllNomadAllocations(), func(text string, index int) {
		app.appContainerExec(text)
	}).SetDrawFunc(centerWidgetFunc).SetBorder(true)

// define job selection, once an option is selected define task options on the task select widget
// with complete functions that execute container attachment
app.selectJobDropdown.
	SetOptions(app.getNomadJobs(), func(jobId string, index int) {

		app.getNomadTaskGroupsForJob(jobId)

		taskGroups := app.getNomadTaskGroupsForJob(jobId)

		if len(taskGroups) == 1 {
			taskGroup := taskGroups[0]
			app.appContainerExec(app.getRandomNomadAllocation(jobId, taskGroup))
		} else {
			app.selectTaskGroupDropdown.
				SetOptions(app.getNomadTaskGroupsForJob(jobId), func(taskGroup string, index int) {

					randomAllocation := app.getRandomNomadAllocation(jobId, taskGroup)
					app.appContainerExec(randomAllocation)
				})
			app.pages.SwitchToPage(selectTaskGroupPageName)
		}
	}).SetDrawFunc(centerWidgetFunc).SetBorder(true)

// set draw func on task group selection
app.selectTaskGroupDropdown.SetDrawFunc(centerWidgetFunc).SetBorder(true)

// define reattach modal
app.reattachModal.AddButtons([]string{reconnectButton, quitButton}).
	SetDoneFunc(func(buttonIndex int, buttonLabel string) {
		switch buttonLabel {
		case reconnectButton:
			app.timeoutChannel <- true
			app.appContainerExec(app.lastUsedAllocationID)
		case quitButton:
			app.cleanupAndExit()
		}
	})
```

4. App suspension, queued updates, etc...

```go
func (a *app) appContainerExec(allocationUUID string) {

	a.Application.Suspend(func() {

		errorCode, err := a.containerExec(allocationUUID)

		if err != nil {

			a.lastUsedAllocationID = allocationUUID

			a.QueueUpdate(func() {
				a.reattachModal.SetText(fmt.Sprintf(reattachModalText, errorCode, err))
				a.pages.SwitchToPage(reattachModalPageName)

        // timeout handling
			})
		} else {
			a.cleanupAndExit()
		}
	})
}
```

5. Draw function meant to center widgets which are not normally centered, like Dropdowns

```go
func centerWidgetFunc(screen tcell.Screen, x int, y int, width int, height int) (int, int, int, int) {
	centerY := y + height/2
	centerX := x + width/3
	return centerX + 1, centerY - 1, width - 2, height - (centerY + 1 - y)
}
```

For more cli libraries in go checkout [awesome-go's command line section](https://awesome-go.com/#command-line).

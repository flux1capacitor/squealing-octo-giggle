# squealing-octo-giggle
Dancing queen
Always
Watching
All
Of
Your
Lives
Key 
Word
**Killbot conundrum**


StudentGrades = Ext.extend(Object, {
    constructor: function (config) {
        Ext.apply(this, config, {
            rowSelectedClass: 'highlight_color',
            cellSelectedClass: "highlight_color",
            mainPanel: null,
            viewport: null
        });

        StudentGrades.superclass.constructor.call(this, config);

        this.rowsCurrentlySaving = [];
        this.studentItemGradingWindow = null;
        this.currentItemId = null;
        this.highlightedRow = null;
        this.itemCurrentlyGrading = null;
        this.PeriodParams = null;
        this.previousStatusBarSaveCount = 0;
        this.statusTimer = null;
        this.changeDateWindow = null;
        this.gradeScaleWindow = null;
        this.weightScaleWindow = null;

        this.quickEditLayer = null;
        this.quickEditPanel = null;
        this.highlightedCell = null;
        this.whatifcurrentitemid = null;
        this.lastGradeTable = null;
        this.codeScroll = null;
        this.sortBySyllabus = false;
        this.sortBySyllabusOrder = null;
        this.sortByDefaultOrder = null;

        this.createLayout();

        if (typeof (FRAME_API) != 'undefined') {
            var enrollmentId = this.EnrollmentId;
            if (!this.isStudent) {
                enrollmentId = Ext.isEmpty(this.SectionId) ? this.CourseId : this.SectionId;
            }
            FRAME_API.setComponentState(this.isObjectives ? 'objectivesummary' : 'studentgrades', this.componentId, {
                courseId: this.CourseId,
                enrollmentId: enrollmentId,
                studentEnrollmentId: this.EnrollmentId,
                sectionId: this.SectionId,
                pageTitle: this.pageTitle,
                subTitle1: this.subTitle1,
                subTitle2: this.subTitle2,
                helpToken: 'STUDENT_VIEWING_COURSE_GRADES_'
            });

            FRAME_API.addListener('viewerclosed', this.viewerClosed, this);
            Ext.EventManager.on(window, 'unload', function () {
                FRAME_API.removeListener('viewerclosed', this.viewerClosed, this);
            }, this);
        }

        //Hook up events
        if (!this.whatIfMode) {
            this.mainPanel.el.select('tr[class*=grade_detail_row]').each(function (el) {
                var itemId = el.getAttribute('data-itemid');
                el.on('mouseover', this.addHighlight.createDelegate(this, [el.dom]));
                el.on('mouseout', this.removeHighlight.createDelegate(this, [el.dom]));
                el.on('click', this.gradeItem.createDelegate(this, [itemId, el.dom]));
                var td = Ext.get('grade_' + this.componentId + '_' + encodeURIComponent(itemId));
                if (td != null) {
                    td.on('click', this.gradeItem.createDelegate(this, [itemId, td.dom]));
                }
            }, this);
        }

        var sortLink = Ext.get('gradesTableSort_' + this.componentId);
        sortLink.on('click', this.toggleSortBySyllabusOrder.createDelegate(this, [sortLink]));

        this.mainPanel.el.select('td[class*=whatif-cell]').each(function (el) {
            var itemId = el.getAttribute('data-itemid');
            el.on('click', this.whatIf.createDelegate(this, [itemId, el.dom]));
        }, this);
        window.onbeforeunload = this.beforeUnload.createDelegate(this);

        if (!this.whatIfMode) {
            if (this.changetarget) {
                this.changeTargetEndDate();
            }
            if (this.showMastery) {
                this.ShowMastery();
            } else {
                document.getElementById("grade_header_grades_" + this.componentId).style.display = "none";
            }
        }
    },

    gotoItem: function (itemid, itemTitle, api) {
        itemid = encodeURIComponent(itemid);
        if (this.hideContent) {
            //call global preview function on FRAME_API
            if (typeof (FRAME_API) != 'undefined') {
                FRAME_API.viewItem(this.EnrollmentId, itemid, itemTitle);
            }
            else if (typeof (showViewWindow) != 'undefined') {
                showViewWindow(this.EnrollmentId, itemid, itemTitle, api);
            }
        }
        else {
            window.location.href = this.frameRoot + "/Component/CoursePlayer?enrollmentid=" + this.EnrollmentId + "&itemid=" + itemid;
        }
    },

    viewerClosed: function () {
        Ext.Ajax.request(Ext.apply({
            url: this.appRoot + '/Gradebook/GradeUpdate.ashx?save_action=Refresh&showpointspossible=true&eid=' + this.EnrollmentId + '&iid=' + encodeURIComponent(this.currentItemId)
        }, this.itemSaving(null, true)));
        this.currentItemId = null;
    },

    ShowGrades: function (e) {
        this.ShowIcon('grades', false);
        this.ShowIcon('objectives', true);
        this.ShowIcon('calculator', true);
        this.ShowIcon('statistics', true);
        this.ShowIcon('scale', true);
        this.ShowIcon('activity', true);
        this.ShowIcon('excused', true);
        this.ShowIcon('lessondetails', true);
        document.getElementById("studentgradesdiv_" + this.componentId).style.display = "";
        document.getElementById("studentmasterydiv_" + this.componentId).style.display = "none";

        var enrollmentId = this.EnrollmentId;
        if (!this.isStudent) {
            enrollmentId = Ext.isEmpty(this.SectionId) ? this.CourseId : this.SectionId;
        }
        FRAME_API.setComponentState('studentgrades', this.componentId, {
            pageTitle: this.isStudent ? this._Grades : this.pageTitle,
            subTitle1: this.subTitle1,
            subTitle2: this.subTitle2,
            courseId: this.CourseId,
            enrollmentId: enrollmentId,
            sectionId: this.SectionId
        });

    },

    ShowMastery: function () {
        this.ShowIcon('grades', !this.hideMastery);
        this.ShowIcon('objectives', false);
        this.ShowIcon('calculator', false);
        this.ShowIcon('statistics', false);
        this.ShowIcon('scale', false);
        this.ShowIcon('activity', !this.hideMastery);
        this.ShowIcon('excused', false);
        this.ShowIcon('lessondetails', false);
        document.getElementById("studentgradesdiv_" + this.componentId).style.display = "none";
        document.getElementById("studentmasterydiv_" + this.componentId).style.display = "";
        var enrollmentId = this.EnrollmentId;
        if (!this.isStudent) {
            enrollmentId = Ext.isEmpty(this.SectionId) ? this.CourseId : this.SectionId;
        }
        var objectiveCombo = Ext.getCmp('objectiveComboId_studentmastery_' + this.componentId);
        if (objectiveCombo) {
            objectiveCombo.setWidth(objectiveCombo.width + 1);
            objectiveCombo.setWidth(objectiveCombo.width);
        }
        FRAME_API.setComponentState('objectivesummary', this.componentId, {
            pageTitle: this.isStudent ? this._Objectives : this.pageTitle,
            subTitle1: this.subTitle1,
            subTitle2: this.subTitle2,
            courseId: this.CourseId,
            enrollmentId: enrollmentId,
            sectionId: this.SectionId
        });
    },

    ShowIcon: function (type, show) {
        var id = String.format("grade_header_{0}_{1}", type, this.componentId);
        var element = document.getElementById(id);
        if (element) element.style.display = show ? '' : 'none';
    },

    convertUtcDateToDate: function (val) {
        if (!Ext.isEmpty(val)) {
            val = val.replace(/\.\d+/, "");
            val = val.replace('Z', '0');
            // see if the date came from the server (Special formatting)
            var date = Date.parseDate(val, 'Y-m-d\\TH:i:sZ');
            if (!date) { // otherwise it came from the client
                date = new Date(val);
            }
            return date;
        }
        return val;
    },

    convertLocalDateToUtcDate: function (date) {
        if (!Ext.isEmpty(date)) {
            return convertLocalTimeToUtcDateTime(date).format('Y-m-d\\TH:i:s\\Z');
        }
        return date;
    },

    createLayout: function () {
        var buttons = [{ xtype: 'tbfill'}];
        if (this.whatIfMode) {
            buttons.push({
                text: this._WhatIfRecalculateButton,
                cls: 'x-btn-text-icon',
                icon: this.appRoot + '/Images/key1_add.png',
                handler: this.whatIfRecalculate,
                scope: this
            });
            buttons.push({
                text: this._Close,
                hidden: this.disableWhatIfSwitch,
                cls: 'x-btn-text-icon',
                icon: this.appRoot + '/Images/scroll2_16.png',
                handler: this.whatIfClose,
                scope: this
            });
        }
        this.mainPanel = new Ext.Panel({
            region: 'center',
            id: 'mainPanel_' + this.componentId,
            autoScroll: true,
            contentEl: 'mainLayout_' + this.componentId,
            buttons: buttons.length > 1 ? buttons : null
        });

        this.viewport = new Ext.Panel({
            renderTo: 'studentgrades_' + this.componentId,
            layout: 'border',
            items: [this.mainPanel]
        });

        this.resizeContainer();
        Ext.EventManager.onWindowResize(this.resizeContainer, this);

        this.mainPanel.getLayoutTarget().on('scroll', this.updateScroll, this);

        if (this.whatIfMode) {
            if (this.mainPanel != null && this.mainPanel.footer != null) {
                this.mainPanel.footer.addClass("color_medium default_border dashboard_list_btns");
            }
        }
    },

    resizeContainer: function () {
        var el = Ext.get('studentgrades_' + this.componentId);
        if (el != null) {
            this.viewport.setSize(el.getSize());
            this.viewport.doLayout();
        }
    },

    beforeUnload: function () {
        var returnString = "";
        if (FRAME_API.showBeforeUnloadPrompts) {
            if (this.itemCurrentlyGrading && this.itemCurrentlyGrading.isModified()) {
                returnString += this._ModifiedButNotSaved;
            }

            if (this.rowsCurrentlySaving.length > 0) {
                returnString += this._ItemsStillSaving;
            }
        }

        if (returnString) return returnString;
    },

    canLeaveCurrentlyGradingItem: function (functionToCallWhenOK) {
        if (this.itemCurrentlyGrading && this.itemCurrentlyGrading.isModified()) {
            Ext.MessageBox.confirm(this._AreYouSure, this._GradeUnsavedDontLeave,
                function (answer) {
                    if (answer == "yes") {
                        this.itemCurrentlyGrading = null;
                        functionToCallWhenOK();
                    }
                }
            , this);
            return;
        }

        if (functionToCallWhenOK) functionToCallWhenOK();
    },

    gradeItem: function (itemId, trElement) {
        this.canLeaveCurrentlyGradingItem((function () {
            this.gradeItemOkToGo(itemId, trElement);
        }).createDelegate(this)
        );
    },

    addHighlight: function (trElement) {
        if (this.highlightedRow == null) {
            Ext.get(trElement).addClass(this.rowSelectedClass);
        }
    },

    removeHighlight: function (trElement) {
        if (this.highlightedRow != trElement) {
            Ext.get(trElement).removeClass(this.rowSelectedClass);
        }
    },

    gradeItemOkToGo: function (itemId, trElement) {
        if (this.highlightedRow != null) {
            Ext.get(this.highlightedRow).removeClass(this.rowSelectedClass);
        }
        Ext.get(trElement).addClass(this.rowSelectedClass);
        this.highlightedRow = trElement;

        this.currentItemId = itemId;

        if (this.isStudent && trElement.cells != null && trElement.cells.length > 0) {
            if (!this.showLinkToItems) {
                return;
            }

            var x = Ext.EventObject.getPageX();
            var titleCell = Ext.get(trElement.cells[0]);
            var titleX = titleCell.getX();
            if (x >= titleX && x <= titleX + titleCell.getWidth()) {
                //We clicked the title area, so open the view window instead of the grade window
                // title is immediately before the font breadcrumb
                this.viewItem(itemId, titleCell.child('div').child('font').dom.previousSibling.nodeValue.trim());
                return;
            }
        }

        var idealHeight = 800;
        var idealWidth = 800;

        var studentItemGradingAreaElement = Ext.get("studentItemGradingArea_" + this.componentId);

        /* If we're already loading something else, abort it and load the new thing instead */
        var currentTransaction = studentItemGradingAreaElement.getUpdater().transaction;
        if (currentTransaction) {
            if (Ext.Ajax.isLoading(currentTransaction)) {
                Ext.Ajax.abort(currentTransaction);
            }
        }

        var pageSizeObj = Ext.get(document.body).getSize();
        var windowHeight = Math.min(pageSizeObj.height / 1.5, idealHeight);
        var windowWidth = Math.min(pageSizeObj.width / 1.5, idealWidth);

        if (this.studentItemGradingWindow == null) {
            this.studentItemGradingWindow = new Ext.Window({
                constrainHeader: true,
                height: windowHeight,
                width: windowWidth,
                layout: 'fit',
                plain: true,
                buttonAlign: 'center',
                items:
                    [
                        {
                            id: 'detailItemPanel_' + this.componentId,
                            xtype: 'panel',
                            layout: 'fit',
                            cls: 'color_light',
                            monitorResize: true
                        }
                    ],
                shim: true,
                modal: false,
                frame: true,
                closable: true,
                closeAction: "hide"
            });

            this.studentItemGradingWindow.on("hide", this.attemptToCloseStudentItemGradingWindow, this);
        }

        this.studentItemGradingWindow.show();

        var isSaving = false;
        for (var i = 0; i < this.rowsCurrentlySaving.length; i++) {
            if (trElement == this.rowsCurrentlySaving[i]) {
                isSaving = true;
                break;
            }
        }

        if (isSaving) {
            Ext.getCmp('detailItemPanel_' + this.componentId).body.update('<div class="loading-indicator">' + this._SavingDotDotDot + '</div>');
        }
        else {
            var requestUrl = this.appRoot + "/Gradebook/Grade.aspx?list=student&showpointspossible=true&lessonasitem=1&enrollmentid=" + this.EnrollmentId + "&itemid=" + encodeURIComponent(itemId) + "&forceteacherview=" + !this.isStudent;
            var enrollment = FRAME_API.findEnrollment(this.EnrollmentId);
            if (enrollment && enrollment.isTestStudent && this.isStudent) {
                requestUrl += '&readonly=1';
            }
            Ext.getCmp('detailItemPanel_' + this.componentId).body.update('<div class="loading-indicator">' + this._LoadingDotDotDot + '</div>');
            studentItemGradingAreaElement.load({ url: requestUrl, scripts: true });
        }
    },

    viewItem: function (itemId, itemTitle) {
        this.currentItemId = itemId;
        this.gotoItem(itemId, itemTitle, {
            closePopup: function () {
                Ext.Ajax.request(Ext.apply({
                    url: this.appRoot + '/Gradebook/GradeUpdate.ashx?save_action=Refresh&showpointspossible=true&eid=' + this.EnrollmentId + '&iid=' + encodeURIComponent(itemId)
                }, this.itemSaving(null, true)));
                this.currentItemId = null;
            }
        });
    },

    attemptToCloseStudentItemGradingWindow: function () {
        if (this.itemCurrentlyGrading && this.itemCurrentlyGrading.isModified()) {
            this.studentItemGradingWindow.show();
        }

        this.canLeaveCurrentlyGradingItem((function () {
            if (this.highlightedRow != null) {
                Ext.get(this.highlightedRow).removeClass(this.rowSelectedClass);
            }
            this.highlightedRow = null;
            this.studentItemGradingWindow.hide();
        }).createDelegate(this));
    },

    //API for grade render?
    setItem: function (newGradingItem) {
        this.itemCurrentlyGrading = newGradingItem;
    },

    getItem: function () { return this.itemCurrentlyGrading; },

    setItemPanel: function (newPanel) {
        var panel = Ext.getCmp('detailItemPanel_' + this.componentId);
        if (panel.items) {
            while (panel.items.length > 0) {
                panel.remove(panel.items.items[0]);
            }
        }

        panel.add(newPanel);
        panel.doLayout();
        this.itemCurrentlyGrading.focus();
    },

    closeItem: function () {
        this.attemptToCloseStudentItemGradingWindow();
    },

    itemSaving: function (item, hideStatus) {
        var savingItemInfo =
            {
                gridRow: this.highlightedRow,
                item: this.itemCurrentlyGrading,
                oldHTML: this.getScoreHTML(this.currentItemId),
                itemId: this.currentItemId
            };

        this.setScoreHTML(this.currentItemId, '<div><img src="' + Ext.BLANK_IMAGE_URL + '" class="gradebook_saving_image"/></div>');

        var additionalOptions = { callback: this.saveFinished, scope: this, savingItem: savingItemInfo };
        additionalOptions.params = "categories=true&returngrade=1&boldscores=1&returndropped=1";
        if (this.PeriodParams) additionalOptions.params += "&" + PeriodParams;

        if (hideStatus != true) {
            this.rowsCurrentlySaving.push(this.highlightedRow);
        }
        this.itemCurrentlyGrading = null;

        //setTimeout(function() { Ext.getCmp('detailItemPanel').body.update('<div class="loading-indicator">' + "Saving..." + '</div>'); }, 10);
        if (this.highlightedRow != null) {
            Ext.get(this.highlightedRow).removeClass(this.rowSelectedClass);
        }
        this.highlightedRow = null;
        if (this.studentItemGradingWindow != null) {
            this.studentItemGradingWindow.hide();
        }

        this.updateDashboardStatus();

        return additionalOptions;
    },

    updateDashboardStatus: function () {
        clearTimeout(this.statusTimer);
        var oldStatusText = this.statusDisplay.getText();
        var newStatusText = "";

        var savingCount = this.rowsCurrentlySaving.length;
        if (savingCount > 0) {
            if (savingCount == 1) {
                newStatusText += String.format(this._Saving1Item);
            }
            else {
                newStatusText += String.format(this._SavingNItems, savingCount);
            }
        }
        else if ((savingCount == 0) &&
                (this.previousStatusBarSaveCount != 0)) {
            newStatusText += this._DoneSaving;
        }

        if (newStatusText) {
            if (newStatusText != oldStatusText) {
                this.statusDisplay.updateStatus(newStatusText);
            }
            this.previousStatusBarSaveCount = savingCount;
            this.statusTimer = this.updateDashboardStatus.defer(3000, this);
        }
        else {
            this.statusDisplay.hide();
        }
    },

    statusDisplay: function () {
        var msgCt;
        var message;
        var isHidden = false;
        var currentText = "";

        function createBox(text) {
            //        return ['<div class="msg" >',
            //                '<div class="x-box-tl"><div class="x-box-tr"><div class="x-box-tc"></div></div></div>',
            //                '<div class="x-box-ml"><div class="x-box-mr"><div class="x-box-mc">', text, '</div></div></div>',
            //                '<div class="x-box-bl"><div class="x-box-br"><div class="x-box-bc"></div></div></div>',
            //                '</div>'].join('');
            return ['<div class="gradebook_statusdisplay highlight_color" >', text, '</div>'].join('');
        }
        return {
            updateStatus: function (text) {
                currentText = text;
                if (!msgCt) {
                    msgCt = Ext.DomHelper.append(document.body, { id: 'msg-div', style: "position:absolute;" }, true);
                }
                var slideIn = false;
                if (!message) {
                    message = Ext.DomHelper.append(msgCt, { html: createBox(text) }, true);
                    slideIn = true;
                    msgCt.alignTo(document, 't-t');
                }
                else {
                    message.update(createBox(text));
                    msgCt.alignTo(document, 't-t');
                    message.frame();
                }

                if (slideIn) message.slideIn('t');
            },
            hide: function () {
                currentText = "";
                if (message) {
                    message.ghost("t", { remove: true });
                    message = null;
                }
            },
            getText: function () {
                return currentText;
            }
        };
    } (),

    itemSaved: function (xmlNode) {
        this.updateScoresFromXML(null, xmlNode, false);
    },

    saveFinished: function (options, success, response) {
        if (success) {
            Ext.util.Format.getIE9PostReturnXml(response);
            if (!response.responseXML || !response.responseXML.firstChild || !response.responseXML.firstChild.firstChild ||
                     (this.getTextFromXMLNode(response.responseXML.firstChild.firstChild) != "OK")) {
                var msg = (response.responseXML && response.responseXML.firstChild && response.responseXML.firstChild.firstChild) ? this.getTextFromXMLNode(response.responseXML.firstChild.firstChild) : null;
                if (msg) {
                    Ext.MessageBox.alert(this._ErrorTitle, String.format(this._UnableToSave, ' ', msg));
                }
                else {
                    this.alertUnexpectedResponse(Ext.apply(options, response, options.savingItem), this._ErrorTypeSaving);
                }
                this.setScoreHTML(options.savingItem.itemId, options.savingItem.oldHTML);
            }
            else {
                this.updateScoresFromXML(options.savingItem.itemId, response.responseXML.firstChild, false);
            }
        }
        else {
            this.alertUnexpectedResponse(Ext.apply(options, response, options.savingItem), this._ErrorTypeSaving);
            this.setScoreHTML(options.savingItem.itemId, options.savingItem.oldHTML);
        }
        this.rowsCurrentlySaving.remove(options.savingItem.gridRow);

        if (this.highlightedRow == options.savingItem.gridRow) {
            this.highlightedRow.onclick();
        }
        this.updateDashboardStatus();
    },

    updateScoresFromXML: function (savingItemId, xmlNode, highlightWhatif) {
        if (savingItemId == null) {
            // savingItemId is null when updating lesson scores, in which case, the score to update could be 
            // any item beneath the lesson. We assume the the <grade> element is always there, and contains the item's ID
            for (var i = 0; i < xmlNode.childNodes.length; i++) {
                if (xmlNode.childNodes[i].nodeType != 1) continue;
                if (xmlNode.childNodes[i].nodeName == "grade") {
                    savingItemId = xmlNode.childNodes[i].getAttribute("id");
                }
            }
        }
        var text, itemId;
        for (var i = 0; i < xmlNode.childNodes.length; i++) {
            if (xmlNode.childNodes[i].nodeType != 1) continue;

            switch (xmlNode.childNodes[i].nodeName) {
                case "grade":
                    this.setScoreHTML(savingItemId, this.getTextFromXMLNode(xmlNode.childNodes[i]));
                    break;
                case "submitted":
                    this.setSubmittedHTML(savingItemId, this.getTextFromXMLNode(xmlNode.childNodes[i]));
                    break;
                case "enrollmentscore":
                    text = this.getTextFromXMLNode(xmlNode.childNodes[i]);
                    text = text.replace('</div>', '').replace('<div>', '');
                    if (highlightWhatif) {
                        text = text.replace('class="', 'class="highlight_color ');
                    }
                    // wrap in parens if it follows course points
                    if (Ext.get("coursePoints_" + this.componentId)) {
                        text = '(' + text + ')';
                    }
                    var courseScoreElement = Ext.get("courseScore_" + this.componentId);
                    if (courseScoreElement) courseScoreElement.update(text);
                    break;
                case "enrollmentpoints":
                    text = this.getTextFromXMLNode(xmlNode.childNodes[i]);
                    text = text.replace('</div>', '').replace('<div>', '');
                    if (highlightWhatif) {
                        text = text.replace('class="', 'class="highlight_color ');
                    }
                    var coursePointsElement = Ext.get("coursePoints_" + this.componentId);
                    if (coursePointsElement) coursePointsElement.update(text);
                    break;
                case "enrollmentcompletion":
                    var courseCompleteElement = Ext.get("grade_label_completed_" + this.componentId);
                    text = this.getTextFromXMLNode(xmlNode.childNodes[i]);
                    if (highlightWhatif) {
                        text = text.replace('<div', '<div class="highlight_color"');
                    }
                    if (courseCompleteElement) courseCompleteElement.update(text);
                    // update the chart, too
                    var chart = Ext.get('grade_label_completedgraph_' + this.componentId);
                    if (chart) {
                        var percent = xmlNode.childNodes[i].getAttribute("percent");
                        if (Ext.isEmpty(percent)) {
                            chart.update("<img src='" + this.appRoot + "/Controls/Chart.ashx?type=empty' />");
                        } else {
                            percent = parseInt(percent);
                            chart.update(String.format("<img src='{0}/Controls/Chart.ashx?type=hbar&height=10&width=200&item0={1}&item1={2}&color0=%2332CD32&color1=White' alt='{1}' />", this.appRoot, percent, 100 - percent));
                        }
                    }
                    break;
                case "categoryhtml":
                    var categoryId = xmlNode.childNodes[i].getAttribute("id");
                    if (categoryId.indexOf('!--LES-') == 0) {
                        categoryId = categoryId.substr('!--LES-'.length);
                        categoryId = categoryId.substr(0, categoryId.length - '--!'.length);
                        this.setScoreHTML(categoryId, this.getTextFromXMLNode(xmlNode.childNodes[i]));
                    }
                    break;
                case "categoryscore":
                    text = this.getTextFromXMLNode(xmlNode.childNodes[i]);
                    var categoryId = xmlNode.childNodes[i].getAttribute("id");
                    // wrap in parens if it follows course points
                    if (Ext.get("cat_points_" + this.componentId + "_" + categoryId) && !Ext.isEmpty(text)) {
                        text = '(' + text + ')';
                    }
                    var categoryScoreElement = Ext.get("cat_score_" + this.componentId + "_" + categoryId);
                    if (categoryScoreElement) categoryScoreElement.update(text);
                    break;
                case "categorypoints":
                    var categoryId = xmlNode.childNodes[i].getAttribute("id");
                    var categoryPointsElement = Ext.get("cat_points_" + this.componentId + "_" + categoryId);
                    if (categoryPointsElement) categoryPointsElement.update(this.getTextFromXMLNode(xmlNode.childNodes[i]));
                    break;
                case "periodscore":
                    var periodId = xmlNode.childNodes[i].getAttribute("id");
                    var periodScoreElement = Ext.get("period_score_" + this.componentId + "_" + periodId);
                    if (periodScoreElement) periodScoreElement.update(this.getTextFromXMLNode(xmlNode.childNodes[i]));
                    break;
                case "dropped":
                    this.removeAllDropped();
                    for (var j = 0; j < xmlNode.childNodes[i].childNodes.length; j++) {
                        var dropId = this.getTextFromXMLNode(xmlNode.childNodes[i].childNodes[j]);
                        this.setRowDropped(dropId);
                    }
                    break;
                case "passrequired":
                    this.setRequiredImage(savingItemId, this.getTextFromXMLNode(xmlNode.childNodes[i]));
                    break;
                default:
                    break;
            }
        }
    },

    setRequiredImage: function (itemId, cls) {
        var td = Ext.get('grade_' + this.componentId + '_' + itemId);
        if (td) {
            var tr = td.parent();
            if (tr != null) {
                var image = tr.child('img');
                if (image) {
                    // remove any existing image, and add the one specified
                    image.removeClass(['grades_required_image_incomplete', 'grades_required_image_passed', 'grades_required_image_failed', 'grades_not_dropped_score_image']);
                    // dropped-score image takes precedence, so don't override it
                    if (!image.hasClass('grades_dropped_score_image')) {
                        image.addClass(cls);
                    }
                }
            }
        }
    },

    setRowDropped: function (itemId) {
        var td = Ext.get('grade_' + this.componentId + '_' + itemId);
        if (td) {
            var tr = td.parent();
            if (tr != null) {
                // add the strikethrough class to this row
                tr.addClass('grades_dropped_score_row');
                var image = tr.child('img');
                if (image) {
                    // make the image visible
                    image.replaceClass('grades_not_dropped_score_image', 'grades_dropped_score_image');
                }
            }
        }
    },

    removeAllDropped: function () {
        var table = Ext.get('gradesTable_' + this.componentId);
        var trs = table.select('tr.grades_dropped_score_row');
        if (trs) {
            trs.each(function (el, composite, index) {
                // remove the strikethrough class from the row
                el.removeClass('grades_dropped_score_row')
                var image = el.child('img');
                if (image) {
                    // hides the image
                    image.replaceClass('grades_dropped_score_image', 'grades_not_dropped_score_image');
                }
                return true;
            });
        }
    },

    setSubmittedHTML: function (itemId, newHTML) {
        var element = Ext.get("subm_" + this.componentId + '_' + itemId);
        if (element) element.update(newHTML);
    },

    setScoreHTML: function (itemId, newHTML) {
        var element = Ext.get("grade_" + this.componentId + '_' + itemId);
        if (element) element.update(newHTML);
    },

    getScoreHTML: function (itemId, newHTML) {
        var element = Ext.get("grade_" + this.componentId + '_' + itemId);
        if (element) return element.dom.innerHTML;
        return "";
    },

    getTextFromXMLNode: function (node) {
        if (Ext.isIE && (Ext.isIEVer < 10)) return typeof (node.text) == 'undefined' ? node.textContent : node.text;
        return node.textContent;
    },

    alertUnexpectedResponse: function (response, whatWasHappeningText) {
        var responseText;
        var errorText = "<html><script>function showTextInNewWindow(text){var newWindow = window.open('', '', 'height=700,width=800,scrollbars=1');newWindow.document.write(text);newWindow.document.close();}</script><body><table border=1>";
        for (var key in response) {
            errorText += "<tr><td>";
            if (key == "responseText") {
                errorText += Ext.util.Format.htmlEncode(key);
                errorText += "<br/><a href='javascript:void(0);' onclick='showTextInNewWindow(responseText);'>" + this._ShowInNewWindow + "</a>";
                responseText = response[key];
            }
            else {
                errorText += Ext.util.Format.htmlEncode(key);
            }
            errorText += "</td>";
            try {
                var newCell = "<td>" + Ext.util.Format.htmlEncode(response[key]) + "</td></tr>";
                errorText += newCell;
            }
            catch (err) {
                // couldn't format some object as a string...skip it and keep going
                errorText += "<td>" + Ext.util.Format.htmlEncode("<display not supported>") + "</td></tr>";
            }
        }
        errorText += "</table></body></html>";

        var id = Math.random();
        Ext.MessageBox.alert(this._ErrorTitle, String.format(this._UnexpectedResponse, whatWasHappeningText, "<br/>", "<a href='javascript:void(0);' id='" + id + "'>", "</a>"));
        Ext.get(id + "").dom.onclick = (function () {
            this.showTextInNewWindow(errorText, responseText);
        }).createDelegate(this);
    },

    showTextInNewWindow: function (text, responseText) {
        var newWindow = window.open('', '', 'height=700,width=800,scrollbars=1,toolbar=1,menubar=1,location=1');
        newWindow.document.write(text);
        newWindow.document.close();
        if (responseText) {
            newWindow.responseText = responseText;
        }
    },

    changeTargetEndDate: function (e) {
        if (Ext.isEmpty(this.targetEndDate)) {
            return;
        }

        var window = this.getChangeDateWindow()
        Ext.getCmp('targetEndDate_' + this.componentId).setValue(this.convertUtcDateToDate(this.targetEndDate));
        window.show();
    },

    getChangeDateWindow: function () {
        var targetPicker = new Ext.form.DateField({
        });

        //targetPicker.applyToMarkup('targetEnd');

        if (!this.changeDateWindow) {
            this.changeDateWindow = new Ext.Window({
                el: 'changeDateDialog_' + this.componentId,
                width: 350,
                autoHeight: true,
                forceLayout: true,
                closeAction: 'hide',
                layout: 'fit',
                modal: true,
                items: [{
                    autoHeight: true,
                    layout: 'table',
                    cls: 'admin-form',
                    layoutConfig: { columns: 2 },
                    items: [{
                        xtype: 'label',
                        cls: 'admin_detail_label',
                        text: this._EndDateLabel + ':'
                    }, {
                        xtype: 'label',
                        cls: 'admin_detail_value',
                        text: this.convertUtcDateToDate(this.endDate).format(Date.patterns.ShortDate)
                    }, {
                        xtype: 'label',
                        cls: 'admin_detail_label',
                        text: this._NonPacedEndDateLabel + ':'
                    }, {
                        xtype: 'datefield',
                        id: 'targetEndDate_' + this.componentId,
                        format: Date.patterns.ShortDate,
                        allowBlank: false,
                        selectOnFocus: true,
                        minValue: this.convertUtcDateToDate(this.startDate),
                        maxValue: this.convertUtcDateToDate(this.endDate),
                        listeners: {
                            change: function (field, newVal, oldVal) {
                                //TODO: Send the changed target end date to the server
                            }
                        }
                    }]
                }],
                buttons: [{ xtype: 'tbfill' }, {
                    text: this._OK,
                    handler: this.changeDateOK,
                    scope: this
                }, {
                    text: this._Cancel,
                    handler: function () {
                        this.changeDateWindow.hide();
                    },
                    scope: this
                }]
            });
        }
        return this.changeDateWindow;
    },

    changeDateOK: function () {
        var field = Ext.getCmp('targetEndDate_' + this.componentId);
        var date = field.getValue();
        var limit = this.convertUtcDateToDate(this.endDate);
        date.setHours(limit.getHours());
        date.setMinutes(limit.getMinutes());
        date.setSeconds(limit.getSeconds());
        if (date > limit) {
            Ext.MessageBox.alert(this._ErrorTitle,
                String.format(this._NonPacedEndDateError, limit.format(Date.patterns.ShortDate)));
            return;
        }

        this.getChangeDateWindow().hide();

        var loadingString = '<div class="loading-indicator">&nbsp;</div>';
        Ext.get('grade_label_target_' + this.componentId).update(loadingString);
        for (var i = 0; i < this.items.length; i++) {
            var el = Ext.get('due_' + this.componentId + '_' + this.items[i]);
            if (el != null) {
                el.update(loadingString);
            }
        }

        date = this.convertLocalDateToUtcDate(date);
        Ext.Ajax.request({
            url: this.appRoot + '/Gradebook/ChangeTargetEndDate.ashx',
            params: {
                enrollmentid: this.EnrollmentId,
                targetdate: date
            },
            method: 'POST',
            success: function (request, options) {
                this.targetEndDate = options.params.targetdate;
                var date = this.convertUtcDateToDate(options.params.targetdate);
                Ext.get('grade_label_target_' + this.componentId).update(date.format(Date.patterns.ShortDate));
                var root = eval('(' + request.responseText + ')');
                for (var i = 0; i < root.length; i++) {
                    var el = Ext.get('due_' + this.componentId + '_' + root[i][0]);
                    if (el != null) {
                        el.update(root[i][1]);
                    }
                }
            },
            failure: function (request, options) {
                this.alertUnexpectedResponse(request, this._ErrorTypeChangeEndDate);
            },
            scope: this
        });
    },

    showGradeScale: function () {
        if (this.gradeScaleWindow == null) {
            this.gradeScaleWindow = new Ext.Window({
                buttonAlign: 'right'
                , constrain: true
                , constrainHeader: true
                , closable: true
                , closeAction: "hide"
                , height: 405
                , width: 350
                , resizable: false
                , layout: 'fit'
                , modal: true
                , title: this._GradeScaleTitle
                , items: [{
                    xtype: 'panel'
                    , autoScroll: true
                    //, autoHeight: true
                    , cls: 'color_light'
                    , items: [{
                        xtype: 'panel'
                        , cls: 'grades_weight_scale_header'
                        , html: this.courseTitle
                    }, {
                        contentEl: 'gradeScaleTableId_' + this.componentId
                    }]
                    , buttons: [{ xtype: 'tbfill' }, {
                        text: this._Close
                        , handler: function () {
                            this.gradeScaleWindow.hide();
                        }
                        , scope: this
                    }]
                }]
            });
        }
        this.gradeScaleWindow.show();
        // style buttons like rest of the app
        this.gradeScaleWindow.items.get(0).footer.addClass("color_medium default_border dashboard_list_btns");
    },

    contactInstructor: function (e) {
        var height = Math.min(400, screen.height);
        var width = Math.min(600, screen.width);
        var features = 'toolbar=no,location=no,directories=no,status=no,menubar=no,scrollbars=yes,resizable=yes,height=' + height + ',width=' + width;
        var newWindow = window.open(this.appRoot + "/Learn/Email.aspx?enrollmentid=" + this.EnrollmentId + "&appendcontext=1", "", features);
        newWindow.focus();
    },

    showWeightScale: function () {
        if (this.weightScaleWindow == null) {
            // parent panel to hold one or both grids
            var parentPanel = new Ext.Panel({
                autoScroll: true
                , cls: 'color_light'
                , items: [{
                    xtype: 'panel'
                        , cls: 'grades_weight_scale_header'
                        , html: this.courseTitle
                }]
                , buttons: [{ xtype: 'tbfill' }, {
                    text: this._Close
                    , handler: function () {
                        this.weightScaleWindow.hide();
                    }
                    , scope: this
                }]
            });
            if (this.periods.length > 0) {
                // create a grid for the period data
                var periodStore = new Ext.data.SimpleStore({
                    fields: ['name', 'weight', 'percent']
                    , data: this.periods
                });
                var periodGrid = new Ext.grid.GridPanel({
                    id: 'periodGridPanel_' + this.componentId
                    , columns: [
                        { id: 'name', header: "<span class='grades_weight_scale_column_header'>" + this._GradingPeriodTitle + "</span>" }
                        , { header: "<span class='grades_weight_scale_column_header'>" + this._Weight + "</span>" }
                        , { header: "<span class='grades_weight_scale_column_header'>%</span>" }
                    ]
                    , autoExpandColumn: 'name'
                    , autoHeight: true
                    , disableSelection: true
                    , enableColumnMove: false
                    , enableHdMenu: false
                    , store: periodStore
                    , width: '100%'
                });
                parentPanel.add(periodGrid);
            }
            if (this.categories.length > 0) {
                // create a grid for the category data
                var categoryStore = new Ext.data.SimpleStore({
                    fields: ['name', 'weight', 'percent']
                    , data: this.categories
                });
                var categoryGrid = new Ext.grid.GridPanel({
                    id: 'catGridPanel_' + this.componentId
                    , columns: [
                        { id: 'name', header: "<span class='grades_weight_scale_column_header'>" + this._GradingCategoryTitle + "</span>" }
                        , { header: "<span class='grades_weight_scale_column_header'>" + this._Weight + "</span>" }
                        , { header: "<span class='grades_weight_scale_column_header'>%</span>" }
                    ]
                    , autoExpandColumn: 'name'
                    , autoHeight: true
                    , disableSelection: true
                    , enableColumnMove: false
                    , enableHdMenu: false
                    , store: categoryStore
                    , style: 'margin-top: 10px'
                    , width: '100%'
                });
                parentPanel.add(categoryGrid);
            }

            this.weightScaleWindow = new Ext.Window({
                buttonAlign: 'right'
                , constrainHeader: true
                , closable: true
                , closeAction: "hide"
                , height: this.periods.length > 0 && this.categories.length > 0 ? 350 : 250
                , width: 500
                , minHeight: 300
                , minWidth: 500
                , layout: 'fit'
                , modal: true
                , title: this._WeightingScaleTitle
                , items: parentPanel
            });
        }
        this.weightScaleWindow.show();
        // style buttons like rest of the app
        this.weightScaleWindow.items.get(0).footer.addClass("color_medium default_border dashboard_list_btns");
    },

    whatIfClose: function () {
        window.location = this.whatIfReturnURL;
    },

    updateScroll: function (s) {
        if (this.codeScroll == null) {
            //Clear out any quick edit stuff
            this.quickEditSubmitGrade(true);
        }
    },

    clearCodeScroll: function () {
        this.codeScroll = null;
    },

    getWhatIfItem: function (itemid) {
        for (var i = 0; i < this.whatifitems.length; i++) {
            if (this.whatifitems[i].itemid == itemid) {
                return this.whatifitems[i];
            }
        }
        return null;
    },
    getWhatIfItemIndex: function (itemid) {
        for (var i = 0; i < this.whatifitems.length; i++) {
            if (this.whatifitems[i].itemid == itemid) {
                return i;
            }
        }
        return NaN;
    },

    whatIf: function (itemId, tdElement) {
        if (this.quickEditLayer != null && this.quickEditLayer.isVisible()) {
            this.quickEditSubmitGrade();
        }

        //    canLeaveCurrentlyGradingItem(function() {
        this.WhatIfGradeItemOkToGo(itemId, tdElement);
        //    }
    },

    WhatIfGradeItemOkToGo: function (itemId, tdElement) {

        //    var studentItemGradingAreaElement = Ext.get("studentItemGradingArea");
        //    var currentTransaction = studentItemGradingAreaElement.getUpdater().transaction;
        //    if (currentTransaction) {
        //        if (Ext.Ajax.isLoading(currentTransaction)) {
        //            //alert("aborting the old request");
        //            Ext.Ajax.abort(currentTransaction);
        //        }
        //    }

        if (this.highlightedCell != null) {
            Ext.get(this.highlightedCell).removeClass(this.cellSelectedClass);
        }
        var highlightedCellEl = Ext.get(tdElement);

        highlightedCellEl.addClass(this.cellSelectedClass);
        this.highlightedCell = tdElement;

        //Scroll the TD into view
        if (this.codeScroll != null) {
            clearTimeout(this.codeScroll);
        }
        this.codeScroll = this.clearCodeScroll.defer(50, this);
        highlightedCellEl.scrollIntoView(this.mainPanel.getLayoutTarget());

        this.whatIfCurrentItemId = itemId;

        //Check to see if it's a quick edit cell
        this.showQuickEditWindow();
    },

    isPassing: function (score, threshold) {
        return this.greaterOrEq(score, threshold);
    },

    // returns a >= b handling rounding errors when a and b are double
    greaterOrEq: function (a, b) {
        // handling rounding errors in double arithmetic; i.e. handle this case: 2.4/3.0 >= .8
        return Math.round(a * 10000) >= Math.round(b * 10000);
    },

    showQuickEditWindow: function () {
        if (this.quickEditLayer != null &&
            this.quickEditLayer.isVisible()) {
            this.quickEditLayer.hide();
        }

        if (this.whatIfCurrentItemId == null) {
            return;
        }

        //    var isSaving = false;
        //    for (var i = 0; i < cellsCurrentlySaving.length; i++) {
        //        if (highlightedCell == cellsCurrentlySaving[i]) {
        //            return;
        //        }
        //    }

        var highlightedCellEl = Ext.get(this.highlightedCell);

        if (this.quickEditLayer == null) {
            this.quickEditLayer = new Ext.Layer({
                shadow: false,
                constrain: false
            });
            this.quickEditLayer.update('<div id="quickEditEl_' + this.componentId + '"></div>');
        }

        if (this.quickEditPanel == null) {
            this.quickEditPanel = new Ext.Panel({
                applyTo: 'quickEditEl_' + this.componentId,
                cls: 'highlight_color',
                bodyStyle: 'height: 26px;',
                layout: 'fit',
                items: [{
                    layout: 'border',
                    items: [{
                        region: 'east',
                        autoWidth: true
                        //                    ,html: '<img onclick="showGradeItemWindow();" ext:qtip="' + '_Details' + '" class="quick-edit-img" src="../Images/document_dirty.png" height="16" width="16"/>'
                    }, {
                        id: 'quickEditEntryPanel_' + this.componentId,
                        region: 'center',
                        layout: 'card',
                        //bodyStyle: 'padding:3px;',
                        activeItem: 0,
                        items: [{
                            layout: 'border',
                            style: 'padding: 3px;',
                            items: [{
                                region: 'east',
                                autoWidth: true,
                                autoHeight: true,
                                html: '<div id="quickEditPointsMax_' + this.componentId + '" style="margin-left:3px;font-weight:bold;">/10000</div>'
                            }, {
                                id: 'quickEditPoints_' + this.componentId,
                                region: 'center',
                                xtype: 'numberfield',
                                style: 'text-align: right;',
                                allowNegative: false,
                                minValue: 0,
                                maxValue: 10000,
                                selectOnFocus: true,
                                enableKeyEvents: true,
                                listeners: {
                                    keydown: {
                                        fn: function (field, e) {
                                            this.quickEditKeyPress(field, e);
                                        },
                                        scope: this
                                    },
                                    valid: {
                                        fn: function (field) {
                                            this.updateQuickEditClass(field);
                                        },
                                        scope: this
                                    }
                                }
                            }]
                        }, {
                            id: 'quickEditPercentEl_' + this.componentId,
                            layout: 'border',
                            style: 'padding: 3px;',
                            items: [{
                                region: 'east',
                                autoWidth: true,
                                autoHeight: true,
                                html: '<div style="margin-left:3px;font-weight:bold;">%</div>'
                            }, {
                                id: 'quickEditPercent_' + this.componentId,
                                region: 'center',
                                xtype: 'numberfield',
                                style: 'text-align: right;',
                                allowNegative: false,
                                minValue: 0,
                                maxValue: 999,
                                selectOnFocus: true,
                                enableKeyEvents: true,
                                listeners: {
                                    keydown: {
                                        fn: function (field, e) {
                                            this.quickEditKeyPress(field, e);
                                        },
                                        scope: this
                                    },
                                    valid: {
                                        fn: function (field) {
                                            this.updateQuickEditClass(field);
                                        },
                                        scope: this
                                    }
                                }
                            }]
                        }, {
                            layout: 'fit',
                            style: 'padding-top: 2px;padding-left: 2px;',
                            items: [{
                                xtype: 'combo',
                                id: 'quickEditGrade_' + this.componentId,
                                cls: 'quick-edit-combo',
                                allowEmpty: true,
                                forceSelection: true,
                                selectOnFocus: true,
                                height: 26,
                                maxHeight: 600,
                                triggerAction: 'all',
                                typeAhead: true,
                                mode: 'local',
                                valueField: 'value',
                                displayField: 'grade',
                                anchor: '100%',
                                store: new Ext.data.Store({
                                    data: [],
                                    reader: new Ext.data.ArrayReader({ id: 0 },
                                        Ext.data.Record.create([{ name: 'grade' }, { name: 'value' }, { name: 'class'}]))
                                }),
                                tpl: '<tpl for="."><div class="x-combo-list-item {class}">{grade}</div></tpl>',
                                enableKeyEvents: true,
                                listeners: {
                                    keydown: {
                                        fn: function (field, e) {
                                            this.quickEditKeyPress(field, e);
                                        },
                                        scope: this
                                    },
                                    select: {
                                        fn: function (combo, r, index) {
                                            this.quickEditDirty = true;
                                            combo.el.removeClass('passing-score');
                                            combo.el.removeClass('failing-score');
                                            combo.el.addClass(r.get('class'));
                                        },
                                        scope: this
                                    }
                                }
                            }]
                        }]
                    }]
                }]
            });
        }

        if (this.studentItemGradingWindow != null &&
            !this.studentItemGradingWindow.hidden) {
            this.studentItemGradingWindow.hide();
        }

        var size = highlightedCellEl.getSize();
        this.quickEditLayer.setSize(size.width - 1, size.height);
        this.quickEditLayer.show();
        this.quickEditLayer.anchorTo(highlightedCellEl, 'tl-tl');
        this.quickEditPanel.doLayout();

        //Choose the correct entry panel and load data
        var entryPanel = Ext.getCmp('quickEditEntryPanel_' + this.componentId);

        var whatifValue = NaN;
        var isValuePercent = null;
        var whatifitem = this.getWhatIfItem(this.whatIfCurrentItemId);
        if (whatifitem != null) {
            whatifValue = whatifitem.value;
            isValuePercent = whatifitem.isValuePercent;
        }
        var weight = this.itemMap[this.whatIfCurrentItemId][3];
        //    var score = points / weight;

        // temporary using Teacher settings
        // TODO: Switch to Student settings
        var gradeEntry = this.itemMap[this.whatIfCurrentItemId][2];
        var passingScore = this.itemMap[this.whatIfCurrentItemId][5];
        switch (gradeEntry) {
            case GradeView.Grade:
                entryPanel.getLayout().setActiveItem(2);
                var grade = Ext.getCmp('quickEditGrade_' + this.componentId);
                var gradeTableId = this.itemMap[this.whatIfCurrentItemId][4];
                var gradeTable = this.gradeTables[gradeTableId];
                var brackets = null;
                if (gradeTable != null) {
                    brackets = gradeTable[1];
                }
                grade.el.removeClass('passing-score');
                grade.el.removeClass('failing-score');
                var percentEntry = (gradeTable == null || gradeTable[0] == GradeScale.Percent);
                //Set the value based on the points/score
                var value = null;


                if (brackets != null && brackets.length > 0 &&
                    !Ext.isEmpty(whatifValue)) {
                    // 100 or weight ????????????????????????????????
                    var tableScore = percentEntry ? whatifValue / 100 : whatifValue;
                    for (var i = 0; i < brackets.length; i++) {
                        var bracketScore = brackets[i][1];
                        var bracketClsScore = percentEntry ? bracketScore : (bracketScore / weight);
                        var cls = this.isPassing(bracketClsScore, passingScore) ? 'passing-score' : 'failing-score';

                        if (brackets[i].length < 3) {
                            brackets[i].push('');
                        }
                        brackets[i][2] = cls;

                        if (value == null && this.greaterOrEq(tableScore, bracketScore)) {
                            value = bracketScore;
                            grade.el.addClass(cls);
                        }
                    }
                }
                if (gradeTableId != this.lastGradeTable) {
                    grade.store.loadData(brackets);
                }
                this.lastGradeTable = gradeTableId;
                grade.doQuery(null, true);
                grade.collapse();
                grade.focus();
                grade.setValue(value);
                grade.selectText();
                break;
            case GradeView.Percent:
                entryPanel.getLayout().setActiveItem(1);
                var number = Ext.getCmp('quickEditPercent_' + this.componentId);
                number.el.removeClass('passing-score');
                number.el.removeClass('failing-score');
                number.el.addClass(this.isPassing((whatifValue / 100), passingScore) ? 'passing-score' : 'failing-score');
                number.setValue(whatifValue);
                number.focus();
                break;
            case GradeView.Points:
            default:
                // update the "out of" number
                Ext.get('quickEditPointsMax_' + this.componentId).update("/" + weight);
                entryPanel.getLayout().setActiveItem(0);
                var number = Ext.getCmp('quickEditPoints_' + this.componentId);
                number.el.removeClass('passing-score');
                number.el.removeClass('failing-score');
                number.el.addClass(this.isPassing((whatifValue / weight), passingScore) ? 'passing-score' : 'failing-score');
                number.setValue(whatifValue);
                number.focus();
                break;
        }
        this.quickEditPanel.doLayout();

        this.quickEditDirty = false;
    },

    quickEditSubmitGrade: function (forceClose) {
        if (this.quickEditLayer == null || !this.quickEditLayer.isVisible()) {
            return;
        }
        if (!this.quickEditDirty) {
            this.closeQuickEditWindow();
            return;
        }
        if (this.whatIfCurrentItemId == null) {
            return;
        }

        var quickScore;
        var gradeEntry = this.itemMap[this.whatIfCurrentItemId][2];
        //    var percentEntry = gradeEntry == GradeView.Percent;
        var isValid = false;
        switch (gradeEntry) {
            case GradeView.Grade:
                var grade = Ext.getCmp('quickEditGrade_' + this.componentId);
                isValid = grade.isValid();
                var raw = grade.getRawValue();
                if (Ext.isEmpty(raw)) {
                    quickScore = '';
                }
                else {
                    quickScore = grade.getValue();
                    if (quickScore == null) {
                        isValid = false;
                        grade.markInvalid();
                    }
                }
                if (!isValid) {
                    if (forceClose) this.closeQuickEditWindow();
                    return;
                }
                percentEntry = this.gradeTables[this.itemMap[this.whatIfCurrentItemId][4]][0] == GradeScale.Percent;
                if (percentEntry && !Ext.isEmpty(quickScore)) {
                    quickScore *= 100.0;
                }
                this.setSubmittedWhatif(percentEntry, gradeEntry, quickScore, raw);
                break;
            case GradeView.Percent:
                var number = Ext.getCmp('quickEditPercent_' + this.componentId);
                isValid = number.isValid();
                if (!isValid) {
                    if (forceClose) this.closeQuickEditWindow();
                    return;
                }
                quickScore = number.getValue();
                this.setSubmittedWhatif(true, gradeEntry, quickScore, quickScore);
                break;
            case GradeView.Points:
            default:
                var number = Ext.getCmp('quickEditPoints_' + this.componentId);
                isValid = number.isValid();
                if (!isValid) {
                    if (forceClose) this.closeQuickEditWindow();
                    return;
                }
                quickScore = number.getValue();
                this.setSubmittedWhatif(false, gradeEntry, quickScore, quickScore);
                break;
        }


        this.closeQuickEditWindow();
    },

    addAttribute: function (name, value) {
        return " " + name + "=\"" + Ext.util.Format.htmlEncode(value) + "\"";
    },

    successWhatIf: function (result, options) {
        this.updateScoresFromXML("", result.responseXML.firstChild, true);
    },


    setSubmittedWhatif: function (isValuePercent, gradeEntry, value, display) {
        if (this.whatIfCurrentItemId == null) {
            return;
        }
        var whatifitem = this.getWhatIfItem(this.whatIfCurrentItemId);
        if (whatifitem == null) {
            return;
        }

        if (Ext.isEmpty(value)) {
            // end-user cleared the grade; so just reset the node to empty
            gradeEntry = GradeView.Grade;   // so '%' doesn't get appended below
            whatifitem.isValuePercent = undefined;
            whatifitem.value = NaN;
        } else {
            whatifitem.isValuePercent = isValuePercent;
            whatifitem.value = value;
        }

        var html = '<div class = "';
        var passingScore = this.itemMap[this.whatIfCurrentItemId][5];
        switch (gradeEntry) {
            case GradeView.Grade:
                var weight = this.itemMap[this.whatIfCurrentItemId][3];
                html += this.isPassing((value / weight), passingScore) ? 'passing-score' : 'failing-score';
                break;

            case GradeView.Percent:
                html += this.isPassing((value / 100.0), passingScore) ? 'passing-score' : 'failing-score';
                break;

            case GradeView.Points:
            default:
                var weight = this.itemMap[this.whatIfCurrentItemId][3];
                html += this.isPassing((value / weight), passingScore) ? 'passing-score' : 'failing-score';
                break;
        }
        html += '">';
        html += display;
        if (gradeEntry == GradeView.Percent) {
            html += '%';
        }
        html += '<//div>';

        var element = Ext.get("grade_" + this.componentId + '_' + this.whatIfCurrentItemId);
        if (element) element.update(html);
    },


    updateQuickEditClass: function (field) {
        if (this.whatIfCurrentItemId == null) {
            return;
        }

        var gradeEntry = this.itemMap[this.whatIfCurrentItemId][2];
        if (gradeEntry != GradeView.Grade) {
            var passingScore = this.itemMap[this.whatIfCurrentItemId][5];
            var value = field.getValue();
            var weight = this.itemMap[this.whatIfCurrentItemId][3];
            field.el.removeClass('passing-score');
            field.el.removeClass('failing-score');
            if (gradeEntry == GradeView.Points) {
                field.el.addClass(this.isPassing((value / weight), passingScore) ? 'passing-score' : 'failing-score');
            }
            else {
                field.el.addClass(this.isPassing((value / 100.0), passingScore) ? 'passing-score' : 'failing-score');
            }
        }
    },

    quickEditKeyPress: function (field, e) {
        if (this.whatIfCurrentItemId == null) {
            return;
        }

        var key = e.getKey();
        var gradeEntry = this.itemMap[this.whatIfCurrentItemId][2];
        var combo = null;
        if (gradeEntry == GradeView.Grade) {
            combo = field;
        }

        if (!Ext.EventObject.isSpecialKey()) {
            this.quickEditDirty = true;
        }
        switch (key) {
            case Ext.EventObject.UP:
            case Ext.EventObject.DOWN:
            case Ext.EventObject.ENTER:
            case Ext.EventObject.RETURN:
                {
                    if (combo == null ||
                    (!combo.isExpanded() && key != Ext.EventObject.DOWN)) {

                        var nextRow = this.getWhatIfItemIndex(this.whatIfCurrentItemId);
                        nextRow = (key == Ext.EventObject.UP) ? nextRow - 1 : nextRow + 1;
                        if (nextRow >= 0 && nextRow < this.whatifitems.length) {
                            field.blur();
                            var itemId = this.whatifitems[nextRow].itemid;
                            var element = Ext.get("grade_" + this.componentId + '_' + itemId);
                            this.whatIf.defer(10, this, [itemId, element]);
                        }
                        else if (this.quickEditDirty) {
                            field.blur();
                            this.quickEditSubmitGrade();
                        }
                        e.preventDefault();
                    }
                    break;
                }
            case Ext.EventObject.ESC:
                if (combo == null || !combo.isExpanded()) {
                    field.blur();
                    this.closeQuickEditWindow();
                    e.preventDefault();
                }
                break;
            case Ext.EventObject.DELETE:
            case Ext.EventObject.BACKSPACE:
                this.quickEditDirty = true;
                break;
            default:
                break;
        }
    },

    closeQuickEditWindow: function () {
        if (this.quickEditLayer != null &&
            this.quickEditLayer.isVisible()) {
            this.quickEditLayer.hide();
            if (this.highlightedCell != null) {
                Ext.get(this.highlightedCell).removeClass(this.cellSelectedClass);
            }
            this.whatIfCurrentItemId = null;
        }
    },


    whatIfRecalculate: function () {
        this.quickEditSubmitGrade();

        var data = ["<grades>"];
        for (var i = 0; i < this.whatifitems.length; i++) {
            data.push("<grade");
            data.push(this.addAttribute("isvaluepercent", this.whatifitems[i].isValuePercent));
            data.push(this.addAttribute("id", this.whatifitems[i].itemid));
            data.push(this.addAttribute("value", this.whatifitems[i].value));
            data.push("/>");
        }
        data.push("</grades>");

        xml = data.join("");

        Ext.Ajax.request({
            url: this.appRoot + '/Gradebook/WhatIfUpdate.ashx?forceteacherview=' + !this.isStudent,
            method: 'POST',
            xmlData: xml,
            success: this.successWhatIf,
            scope: this,
            params: {
                eid: this.EnrollmentId
            }
        });
    },

    toggleSortBySyllabusOrder: function (sortLink) {
        var order = [];
        var gradesTable = Ext.get('gradesTable_' + this.componentId);
        if (this.sortBySyllabus) {
            sortLink.update(this._SortBySyllabus);
            order = this.sortByDefaultOrder;

            this.sortBySyllabus = false;
        } else {
            sortLink.update(this._SortByPeriodsAndCategories);

            if (this.sortBySyllabusOrder == null) {
                this.populateSortOrders(gradesTable);
            }
            order = this.sortBySyllabusOrder;

            this.sortBySyllabus = true;
        }

        var tableBody
...

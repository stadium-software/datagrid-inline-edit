# Datagrid Inline Editing

Allowing users to conveniently and swiftly edit entire DataGrids or DataGrid rows

https://github.com/stadium-software/datagrid-inline-edit/assets/2085324/63e775db-769e-4361-a242-6094a8d031f6

## Sample applications
This repo contains one Stadium 6.7 application
[DataGridInlineEditing.sapz](Stadium6/DataGridInlineEditing.sapz?raw=true)

# Content
- [Datagrid Inline Editing](#datagrid-inline-editing)
  - [Sample applications](#sample-applications)
- [Content](#content)
- [Version](#version)
- [Common Setup](#common-setup)
  - [Application Setup](#application-setup)
  - [Database, Connector and DataGrid](#database-connector-and-datagrid)
  - [Type Setup](#type-setup)
- [DataGrid Editing](#datagrid-editing)
  - [DataGrid Editing Global Script Setup](#datagrid-editing-global-script-setup)
  - [DataGrid Editing Page-Script Setup](#datagrid-editing-page-script-setup)
  - [DataGrid Editing Page Setup](#datagrid-editing-page-setup)
  - [DataGrid Editing Page.Load Event Setup](#datagrid-editing-pageload-event-setup)
  - [DataGrid Editing Button.Click Event Setup](#datagrid-editing-buttonclick-event-setup)
- [Row Editing](#row-editing)
  - [Row Editing Global Script Setup](#row-editing-global-script-setup)
  - [Row Editing Page-Script Setup](#row-editing-page-script-setup)
  - [Row Editing Page Setup](#row-editing-page-setup)
  - [Row Editing Page.Load Event Setup](#row-editing-pageload-event-setup)
  - [Row Editing Edit.Click Event Setup](#row-editing-editclick-event-setup)
- [Styling](#styling)
  - [Applying the CSS](#applying-the-css)
  - [Customising CSS](#customising-css)
  - [CSS Upgrading](#css-upgrading)

# Version 
1.1

**DataGrid Editing**
1. Added code to return error when ID column is not found
2. Added code to skip rules that do not have corresponding columns
3. Added code to detect uniqueness of DataGrid class on page

**Row Editing**
1. Added code to skip rules that do not have corresponding columns
2. Bug fix: Empty column headings caused save button to appear in the wrong column
3. Added code to detect uniqueness of DataGrid class on page

# Common Setup

## Application Setup
1. Check the *Enable Style Sheet* checkbox in the application properties

## Database, Connector and DataGrid
1. Use the instructions from [this repo](https://github.com/stadium-software/samples-database) to setup the database and DataGrid for this sample

## Type Setup
1. Create a *Type* called "FormField"
2. Add the following properties to the type
   1. "name" (Any)
   2. "type" (Any)
   3. "required" (Any)
   4. "min" (Any)
   5. "max" (Any)
   6. "data" (List)
      1. "Item" (Any)

![Form Fields](images/FormFieldType.png)

# DataGrid Editing
For this module to work, the page must have a *DataGrid* and a *Button* that users use to enable the edit functionality

## DataGrid Editing Global Script Setup
1. Create a Global Script called "EditableDataGrid"
2. Add four input parameters to the Global Script
   1. ButtonClassName
   2. DataGridClass
   3. FormFields
   4. IdentityColumnHeader
3. Drag a *JavaScript* action into the script
4. Add the Javascript below into the JavaScript code property (ignore the validation error message "Invalid script was detected")
```javascript
let scope = this;
let dgClassName = "." + ~.Parameters.Input.DataGridClass;
let dg = document.querySelectorAll(dgClassName);
if (dg.length == 0) {
    dg = document.querySelector(".data-grid-container");
} else if (dg.length > 1) {
    console.error("The class '" + dgClassName + "' is assigned to multiple DataGrids. DataGrids using this script must have unique classnames");
    return false;
} else { 
    dg = dg[0];
}
dg.classList.add("stadium-inline-edit-datagrid");
let table = dg.querySelector("table");
let rowFormFields = ~.Parameters.Input.FormFields;
let IDColumn = ~.Parameters.Input.IdentityColumnHeader;
let buttonParentClass = ~.Parameters.Input.ButtonClassName;
let stadiumButtonClass = "stadium-inline-edit-datagrid-button";
let editButtonParent = document.querySelector("." + buttonParentClass);
let editButton;
if (!editButtonParent) {
    editButtonParent = document.createElement("div");
    editButton = document.createElement("button");
    editButtonParent.classList.add(stadiumButtonClass);
    dg.prepend(editButtonParent);
} else { 
    editButtonParent.classList.add(stadiumButtonClass);
    editButton = editButtonParent.querySelector("button");
}
let buttonBar = document.createElement("div");
let rows = table.querySelectorAll("tbody tr");
let IDColNo;
let options = {
    characterData: true,
    attributes: false,
    childList: true,
    subtree: true,
    characterDataOldValue: true,
},
observer = new MutationObserver(resetDataGrid);

function initForm(){
    let formFields = enrichFormFields(rowFormFields);
    let result = formFields.find((obj) => {
        return obj.type == "Identity";
    });
    if (!result) {
        console.error("The identity column was not found");
        return false;
    }
    IDColNo = result.colNo;

    let form = document.querySelector(".datagrid-inline-edit-form");
    if (!form) {
        let container = document.querySelector('div.container');
        form = document.createElement('form');
        form.classList.add("datagrid-inline-edit-form");
        container.parentNode.insertBefore(form, container);
        form.appendChild(container);
        form.addEventListener("submit", saveButtonClick);
    }

    for (let j = 0; j < rows.length; j++){
        for (let i = 0; i < formFields.length; i++) {
            let colNo = formFields[i].colNo;
            if (!colNo) continue;
            let name = formFields[i].name;
            let value = rows[j].querySelector("td:nth-child(" + formFields[i].colNo + ")").innerText;
            let type = formFields[i].type;
            let data = formFields[i].data;
            let min = formFields[i].min;
            let max = formFields[i].max;
            let required = formFields[i].required;
            let el;
            rows[j].classList.remove("edited");
            rows[j].addEventListener("click", function () { 
                let editing = document.querySelector(".editing");
                if (editing) {
                    editing.classList.remove("editing");
                }
                rows[j].classList.add("editing");
            });
            if (type == "Identity") {
                el = document.createElement("input");
                el.value = value;
                el.classList.add("visually-hidden");
                el.setAttribute("stadium-form-name", name);
            } else if (type == "date") {
                el = document.createElement("input");
                el.setAttribute("type", "date");
                el.classList.add("form-control");
                if (min) {
                    let dmin = new Date(min);
                    min = dmin.getFullYear() + '-' + ('0' + (dmin.getMonth() + 1)).slice(-2) + '-' + ('0' + dmin.getDate()).slice(-2);
                    el.setAttribute("min", min);
                }
                if (max) {
                    let dmax = new Date(max);
                    max = dmax.getFullYear() + '-' + ('0' + (dmax.getMonth() + 1)).slice(-2) + '-' + ('0' + dmax.getDate()).slice(-2);
                    el.setAttribute("max", max);
                }
                el.setAttribute("stadium-form-name", name);
                let d = new Date(value);
                el.value = d.getFullYear() + '-' + ('0' + (d.getMonth() + 1)).slice(-2) + '-' + ('0' + d.getDate()).slice(-2);
            } else if (type == "number") {
                el = document.createElement("input");
                el.setAttribute("type", "number");
                if (min) el.setAttribute("min", min);
                if (max) el.setAttribute("max", max);
                el.setAttribute("onkeydown", "return event.keyCode !== 69");
                el.value = value;
                el.setAttribute("stadium-form-name", name);
                el.classList.add("form-control");
            } else if (type == "checkbox") {
                el = document.createElement("input");
                el.setAttribute("stadium-form-name", name);
                el.setAttribute("type", "checkbox");
                if (value == "true" || value == "Yes" || value == "1") {
                    el.setAttribute("checked", "");
                }
            } else if (type == "dropdown") {
                el = document.createElement("select");
                for (let j = 0; j < data.length; j++) {
                    let option = document.createElement("option");
                    option.text = data[j];
                    option.value = data[j];
                    el.appendChild(option);
                }
                el.value = value;
                el.setAttribute("stadium-form-name", name);
                el.classList.add("form-control");
            } else { 
                el = document.createElement("input");
                el.value = value;
                el.setAttribute("stadium-form-name", name);
                el.setAttribute("type", type);
                el.classList.add("form-control");
            }
            if (el) {
                el.classList.add("stadium-inline-form-control");
                let cell = rows[j].querySelector("td:nth-child(" + colNo + ")");
                if (type != "Identity") cell.querySelector("div").classList.add("visually-hidden");
                if (required) el.setAttribute("required", "");
                cell.appendChild(el);
            }
        }
    }
    prepareDataGrid();
}
document.onkeydown = function (evt) {
    evt = evt || window.event;
    var isEscape = false;
    if ("key" in evt) {
        isEscape = (evt.key === "Escape" || evt.key === "Esc");
    } else {
        isEscape = (evt.keyCode === 27);
    }
    if (isEscape) {
        resetDataGrid();
    }
};

initForm();
observer.observe(dg, options);

/*--------------------------------------------------------------------------------------*/

function prepareDataGrid() { 
    buttonBar.classList.add("edit-button-bar");
    editButton.classList.add("visually-hidden");
    
    let saveButton = document.createElement("button");
    let cancelLink = document.createElement("a");
    saveButton.setAttribute("class", "btn btn-lg btn-default");
    saveButton.setAttribute("type", "submit");
    saveButton.innerText = "Save";

    cancelLink.setAttribute("class", "btn btn-lg btn-link");
    cancelLink.innerText = "Cancel";
    cancelLink.addEventListener("click", resetDataGrid);
    
    buttonBar.appendChild(saveButton);
    buttonBar.appendChild(cancelLink);
    editButtonParent.appendChild(buttonBar);

    let pagination = document.querySelector(".pagination");
    if (pagination) pagination.classList.add("visually-hidden");
    let datagridheader = document.querySelector(".data-grid-header");
    if (datagridheader) datagridheader.classList.add("visually-hidden");

    let arrHeadings = table.querySelectorAll("thead th a");
    for (let i = 0; i < arrHeadings.length; i++) {
        arrHeadings[i].classList.add("visually-hidden");
        let heading = document.createElement("span");
        heading.innerText = arrHeadings[i].innerText;
        heading.classList.add("inline-edit-heading");
        arrHeadings[i].parentElement.appendChild(heading);
     }
}
function saveButtonClick(e) { 
    e.preventDefault();
    let arrGridData = [];
    for (let j = 0; j < rows.length; j++) {
        let formFields = rows[j].querySelectorAll("[stadium-form-name]");
        let arrData = [];
        for (let i = 0; i < formFields.length; i++) {
            let fieldValue = formFields[i].value;
            if (formFields[i].getAttribute("type") == "checkbox") fieldValue = formFields[i].checked;
            let field = { Name: formFields[i].getAttribute("stadium-form-name"), Value: fieldValue };
            arrData.push(field);
        }
        arrGridData.push(arrData);
    }
    resetDataGrid();
    scope.SaveGrid(arrGridData);
}
function enrichFormFields(data) {
    let arrHeadings = table.querySelectorAll("thead th");
    for (let i = 0; i < arrHeadings.length; i++) {
        let heading = arrHeadings[i].querySelector("a");
        if (heading) {
            let needle = heading.textContent.toLowerCase().replaceAll(" ","");
            let index = data.findIndex((col) => col.name.toLowerCase().replaceAll(" ","") == needle);
            if (index > -1) {
                data[index].colNo = i + 1;
            } else if (IDColumn.toLowerCase().replaceAll(" ","") == arrHeadings[i].innerText.toLowerCase().replaceAll(" ","")) {
                data.push({ name: IDColumn, colNo: i + 1, type: "Identity" });
            }
        }
    }
    return data;
}
function resetDataGrid() { 
    observer.disconnect();
    let editing = document.querySelector(".editing");
    if (editing) {
        editing.classList.remove("editing");
    }
    let allFields = table.querySelectorAll(".stadium-inline-form-control");
    for (let i = 0; i < allFields.length; i++) { 
        allFields[i].remove();
    }
    let visuallyHidden = dg.querySelectorAll(".visually-hidden");
    for (let i = 0; i < visuallyHidden.length; i++) { 
        visuallyHidden[i].classList.remove("visually-hidden");
    }
    editButton.classList.remove("visually-hidden");
    buttonBar.remove();
    let arrHeadings = table.querySelectorAll(".inline-edit-heading");
    for (let i = 0; i < arrHeadings.length; i++) {
        arrHeadings[i].remove();
     }
}
```

## DataGrid Editing Page-Script Setup
1. Create a Script inside of the Page called "SaveGrid"
2. Add one input parameter to the Script
   1. GridData
3. Drag a *Notification* action into the script
4. In the *Message* property, select the *GridData* parameter from the *Script Input Parameters* category

## DataGrid Editing Page Setup
1. Drag a *Button* control into the page and enter some text (e.g. Edit DataGrid)
2. Add a class of your choosing to the *Button* *Classes* property to uniquely identify this button (e.g. datagrid-inline-edit-button)
3. Drag a *DataGrid* control to the page ([see above](#database-connector-and-datagrid))
4. Add a class of your choosing to the *DataGrid* *Classes* property (e.g datagrid-inline-edit)
5. Note: If multiple editable DataGrids are shown on one page, each DataGrid and each corresponding edit button must have unique classnames

## DataGrid Editing Page.Load Event Setup
1. Populate the DataGrid with data ([see above](#database-connector-and-datagrid))

## DataGrid Editing Button.Click Event Setup
1. Drag a *List* action into the event script and name the List "FormFields"
2. Set the List *Item Type* property to "Types.FormField"
3. Define the editable columns of your datagrid and their form fields
   1. name: The heading of the column
   2. type: The type of the column. Supported are
      1. text
      2. date
      3. number
      4. checkbox
      5. dropdown
      6. email
   3. required: A boolean (add "true" if required)
   4. min: A minimum value for number or date columns
   5. max: A maximum value for number or date columns
   6. data: A simple list of values for dropdowns (see example below)
```json
= [{
	"name": "End Date",
	"type": "date",
	"required": "true"
},{
	"name": "First Name",
	"type": "text"
},{
	"name": "Last Name",
	"type": "text"
},{
	"name": "No of Children",
	"type": "number",
	"min": "0",
	"max": "10",
	"required": "true"
},{
	"name": "No of Pets",
	"type": "number",
	"min": "0",
	"max": "10",
	"required": "true"
},{
	"name": "Start Date",
	"type": "date",
	"min": "01-01-2010",
	"max": "01-01-2024"
},{
	"name": "Healthy",
	"type": "checkbox"
},{
	"name": "Happy",
	"type": "checkbox"
},{
	"name": "Subscription",
	"type": "dropdown",
	"data": ["", "Subscribed", "Unsubscribed", "No data"],
	"required": "true"
}]
```
4. Drag the Global Script called "EditableDataGrid" into the event script
5. Complete the Input properties for the script
   1. ButtonClassName: The unique classname you assigned to the *Button* control (e.g. datagrid-inline-edit-button)
   2. DataGridClass: The unique classname you assigned to the *DataGrid* (e.g datagrid-inline-edit)
   3. FormFields: Select the *List* called "FormFields" from the dropdown
   4. IdentityColumnHeader: The header of the column that uniquely identifies each row (e.g. ID)

![Inline Editing Input Parameters](images/InlineEditingInputParameters.png)

# Row Editing
For this module to work, the DataGrid must contain an Edit column and the Edit column must have a click event handler

## Row Editing Global Script Setup
1. Create a Global Script called "EditableRow"
2. Add five input parameters to the Global Script
   1. DataGridClass
   2. EditColumnHeader
   3. FormFields
   4. IdentityColumnHeader
   5. IdentityValue
3. Drag a *JavaScript* action into the script
4. Add the Javascript below into the JavaScript code property (ignore the validation error message "Invalid script was detected")
```javascript
let scope = this;
let dgClassName = "." + ~.Parameters.Input.DataGridClass;
let dg = document.querySelectorAll(dgClassName);
if (dg.length == 0) {
    dg = document.querySelector(".data-grid-container");
} else if (dg.length > 1) {
    console.error("The class '" + dgClassName + "' is assigned to multiple DataGrids. DataGrids using this script must have unique classnames");
    return false;
} else { 
    dg = dg[0];
}
dg.classList.add("stadium-inline-edit-datagrid");
let table = dg.querySelector("table");
let rowData = ~.Parameters.Input.FormFields;
let IDColumn = ~.Parameters.Input.IdentityColumnHeader;
let IDValue = ~.Parameters.Input.IdentityValue;
let EditColumn = ~.Parameters.Input.EditColumnHeader;
let IDColNo, rowNumber;
let options = {
    characterData: true,
    attributes: false,
    childList: true,
    subtree: true,
    characterDataOldValue: true,
},
observer = new MutationObserver(resetDataGridRow);

initForm();
document.onkeydown = function (evt) {
    evt = evt || window.event;
    var isEscape = false;
    if ("key" in evt) {
        isEscape = (evt.key === "Escape" || evt.key === "Esc");
    } else {
        isEscape = (evt.keyCode === 27);
    }
    if (isEscape) {
        resetDataGridRow();
    }
};
observer.observe(dg, options);

/*--------------------------------------------------------------------------------------*/

function initForm() { 
    
    removeForm();
    addForm();

    let enrichedRowData = enrichRowData(rowData);
    let result = enrichedRowData.find((obj) => {
        return obj.type == "Identity";
    });
    if (!result) {
        console.error("The identity column was not found");
        return false;
    }
    IDColNo = result.colNo;

    let IDCells = table.querySelectorAll("tbody tr td:nth-child(" + IDColNo + ")");
    for (let i = 0; i < IDCells.length; i++) {
        let rowtr = IDCells[i].parentElement;
        if (IDCells[i].innerText == IDValue) {
            rowNumber = i + 1;
        } else { 
            rowtr.classList.add("opacity");
            rowtr.addEventListener("click", resetDataGridRow);
        }
    }

    let row = table.querySelector("tbody tr:nth-child(" + rowNumber + ")");
    row.classList.add("editing");
    for (let i = 0; i < enrichedRowData.length; i++) {
        let colNo = enrichedRowData[i].colNo;
        if (!colNo) continue;
        let name = enrichedRowData[i].name;
        let value = row.querySelector("td:nth-child(" + enrichedRowData[i].colNo + ")").innerText;
        let type = enrichedRowData[i].type;
        let data = enrichedRowData[i].data;
        let min = enrichedRowData[i].min;
        let max = enrichedRowData[i].max;
        let required = enrichedRowData[i].required;
        let el;
        if (type == "Identity") {
            el = document.createElement("input");
            el.value = value;
            el.classList.add("visually-hidden");
            el.setAttribute("stadium-form-name", name);
        } else if (type == "date") {
            el = document.createElement("input");
            el.setAttribute("type", "date");
            el.classList.add("form-control");
            if (min) {
                let dmin = new Date(min);
                min = dmin.getFullYear() + '-' + ('0' + (dmin.getMonth() + 1)).slice(-2) + '-' + ('0' + dmin.getDate()).slice(-2);
                el.setAttribute("min", min);
            }
            if (max) {
                let dmax = new Date(max);
                max = dmax.getFullYear() + '-' + ('0' + (dmax.getMonth() + 1)).slice(-2) + '-' + ('0' + dmax.getDate()).slice(-2);
                el.setAttribute("max", max);
            }
            el.setAttribute("stadium-form-name", name);
            let d = new Date(value);
            el.value = d.getFullYear() + '-' + ('0' + (d.getMonth() + 1)).slice(-2) + '-' + ('0' + d.getDate()).slice(-2);
        } else if (type == "number") {
            el = document.createElement("input");
            el.setAttribute("type", "number");
            if (min) el.setAttribute("min", min);
            if (max) el.setAttribute("max", max);
            el.setAttribute("onkeydown", "return event.keyCode !== 69");
            el.value = value;
            el.setAttribute("stadium-form-name", name);
            el.classList.add("form-control");
        } else if (type == "checkbox") {
            el = document.createElement("input");
            el.setAttribute("stadium-form-name", name);
            el.setAttribute("type", "checkbox");
            if (value == "true" || value == "Yes" || value == "1") {
                el.setAttribute("checked", "");
            }
        } else if (type == "dropdown") {
            el = document.createElement("select");
            for (let j = 0; j < data.length; j++) {
                let option = document.createElement("option");
                option.text = data[j];
                option.value = data[j];
                el.appendChild(option);
            }
            el.value = value;
            el.setAttribute("stadium-form-name", name);
            el.classList.add("form-control");
        } else if (type == "EditLink") {
            el = document.createElement("button");
            el.classList.add("stadium-form-save");
            el.setAttribute("type", "submit");
            el.innerText = "Save";
        } else {
            el = document.createElement("input");
            el.value = value;
            el.setAttribute("stadium-form-name", name);
            el.classList.add("form-control");
        } 
        if (el) {
            el.classList.add("stadium-inline-form-control");
            if (required) el.setAttribute("required", "");
            let cell = row.querySelector("td:nth-child(" + colNo + ")");
            if (type != "Identity") cell.querySelector("div").classList.add("visually-hidden");
            cell.appendChild(el);
        }
    }
}
function enrichRowData(data) {
    let arrHeadings = table.querySelectorAll("thead th a");
    for (let i = 0; i < arrHeadings.length; i++) {
        let index = data.findIndex((col) => col.name.toLowerCase().replaceAll(" ","") == arrHeadings[i].innerText.toLowerCase().replaceAll(" ",""));
        if (index > -1) {
            data[index].colNo = i + 1;
        } else if (IDColumn.toLowerCase() == arrHeadings[i].innerText.toLowerCase()) {
            data.push({ name: IDColumn, colNo: i + 1, type: "Identity" });
        } else if (EditColumn.toLowerCase() == arrHeadings[i].innerText.toLowerCase()) {
            data.push({ name: EditColumn, colNo: i + 1, type: "EditLink" });
        }
    }
    return data;
}
function resetDataGridRow(){ 
    observer.disconnect();
    let allFields = table.querySelectorAll(".stadium-inline-form-control");
    for (let i = 0; i < allFields.length; i++) { 
        allFields[i].remove();
    }
    let visuallyHidden = table.querySelectorAll(".visually-hidden");
    for (let i = 0; i < visuallyHidden.length; i++) { 
        visuallyHidden[i].classList.remove("visually-hidden");
    }
    let rows = table.querySelectorAll("tbody tr");
    for (let i = 0; i < rows.length; i++) { 
        rows[i].classList.remove("opacity");
        rows[i].removeEventListener("click", resetDataGridRow, false);
    }
    removeForm();
}
function saveButtonClick(e) { 
    e.preventDefault();
    e.target.closest("form").removeEventListener("submit", saveButtonClick);
    let editedrow = table.querySelector("tbody tr:nth-child(" + rowNumber + ")");
    let formFields = editedrow.querySelectorAll("[stadium-form-name]");
    let arrData = [];
    for (let i = 0; i < formFields.length; i++) { 
        let fieldValue = formFields[i].value;
        if (formFields[i].getAttribute("type") == "checkbox") fieldValue = formFields[i].checked;
        let field = { Name: formFields[i].getAttribute("stadium-form-name"), Value: fieldValue};
        arrData.push(field);
    }
    scope.SaveRow(arrData);
    resetDataGridRow();
    handleRowStyling(editedrow);
}
function removeForm(){ 
    let formParent = dg.parentElement.parentElement;
    let form = formParent.querySelector(".datagrid-inline-edit-form");
    if (form) {
        formParent.appendChild(dg);
        form.remove();
    }
    let editing = document.querySelector(".editing");
    if (editing) {
        editing.classList.remove("editing");
    }
}
function addForm() { 
    let formParent = dg.parentElement.parentElement;
    let form = formParent.querySelector(".datagrid-inline-edit-form");
    form = document.createElement('form');
    form.classList.add("datagrid-inline-edit-form");
    dg.parentNode.insertBefore(form, dg);
    form.appendChild(dg);
    form.addEventListener("submit", saveButtonClick);
}
function handleRowStyling(el) { 
    let edited = document.querySelectorAll(".edited");
    for (let i = 0; i < edited.length; i++) { 
        edited[i].classList.remove("edited");
    }
    el.classList.add("edited");
    let tm = getComputedStyle(el).getPropertyValue('--row-inline-edit-edited-row-fadeout-speed').replace("s", "");
    setTimeout(function () { 
        el.classList.remove("edited");
    }, tm * 1000);
}
```

## Row Editing Page-Script Setup
1. Create a Script inside of the Page called "SaveRow"
2. Add one input parameter to the Script
   1. RowData
3. Drag a *Notification* action into the script
4. In the *Message* property, select the *RowData* parameter from the *Script Input Parameters* category

## Row Editing Page Setup
1. Drag a *DataGrid* control to the page ([see above](#database-connector-and-datagrid))
2. Add a class of your choosing to the *DataGrid* *Classes* property ti uniquely identify this DataGrid on this page (e.g datagrid-inline-edit)
3. Note: If multiple editable DataGrids are shown on one page, each DataGrid must have a unique classname

## Row Editing Page.Load Event Setup
1. Populate the DataGrid with data ([see above](#database-connector-and-datagrid))

## Row Editing Edit.Click Event Setup
1. Drag a *List* action into the event script and name the List "FormFields"
2. Set the List *Item Type* property to "Types.FormField"
3. Define the editable columns of your datagrid and their form fields
   1. name: The heading of the column
   2. type: The type of the column. Supported are
      1. text
      2. date
      3. number
      4. checkbox
      5. dropdown
      6. email
   3. required: A boolean (add "true" if required)
   4. min: A minimum value for number or date columns
   5. max: A maximum value for number or date columns
   6. data: A simple list of values for dropdowns (see example below)
```json
= [{
	"name": "End Date",
	"type": "date",
	"required": "true"
},{
	"name": "First Name",
	"type": "text"
},{
	"name": "Last Name",
	"type": "text"
},{
	"name": "No of Children",
	"type": "number",
	"min": "0",
	"max": "10",
	"required": "true"
},{
	"name": "No of Pets",
	"type": "number",
	"min": "0",
	"max": "10",
	"required": "true"
},{
	"name": "Start Date",
	"type": "date",
	"min": "01-01-2010",
	"max": "01-01-2024"
},{
	"name": "Healthy",
	"type": "checkbox"
},{
	"name": "Happy",
	"type": "checkbox"
},{
	"name": "Subscription",
	"type": "dropdown",
	"data": ["", "Subscribed", "Unsubscribed", "No data"],
	"required": "true"
}]
```
4. Drag the Global Script called "EditableRow" into the event script
5. Complete the Input properties for the script
   1. DataGridClass: The unique classname you assigned to the *DataGrid*
   2. EditColumnHeader: The header of the Edit column
   3. FormFields: Select the *List* called "FormFields" from the dropdown
   4. IdentityColumnHeader: The header of the column that uniquely identifies each row (e.g. ID)
   5. IdentityValue: The value from the IdentityColumn that uniquely identifies the row

![Inline Editing Input Parameters](images/InlineRowEditingInputParameters.png)

# Styling
Various elements in this module can be styled using the two CSS files in this repo

## Applying the CSS

**Stadium 6.6 or higher**
1. Create a folder called "CSS" inside of your Embedded Files in your application
2. Drag the two CSS files from this repo [*datagrid-inline-edit-variables.css*](datagrid-inline-edit-variables.css) and [*datagrid-inline-edit.css*](datagrid-inline-edit.css) into that folder
3. Paste the link tags below into the *head* property of your application
```html
<link rel="stylesheet" href="{EmbeddedFiles}/CSS/datagrid-inline-edit.css">
<link rel="stylesheet" href="{EmbeddedFiles}/CSS/datagrid-inline-edit-variables.css">
``` 

![](images/ApplicationHeadProp.png)

**Versions lower than 6.6**
1. Copy the CSS from the two css files into the Stylesheet in your application

## Customising CSS
1. Open the CSS file called [*datagrid-inline-edit-variables.css*](datagrid-inline-edit-variables.css) from this repo
2. Adjust the variables in the *:root* element as you see fit
3. Overwrite the file in the CSS folder of your application with the customised file

## CSS Upgrading
To upgrade the CSS in this module, follow the [steps outlined in this repo](https://github.com/stadium-software/samples-upgrading)

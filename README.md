# Datagrid Inline Editing

Allowing users to conveniently and swiftly edit entire DataGrids

https://github.com/stadium-software/datagrid-inline-edit/assets/2085324/4287fc6c-a790-4161-9fd2-c92e1d56da3d

## Sample applications
This repo contains one Stadium 6.7 application
[DataGridInlineEditing.sapz](Stadium6/DataGridInlineEditing.sapz?raw=true)

# Content
- [Datagrid Inline Editing](#datagrid-inline-editing)
  - [Sample applications](#sample-applications)
- [Content](#content)
  - [Version](#version)
- [Setup](#setup)
  - [Application Setup](#application-setup)
  - [Database, Connector and DataGrid](#database-connector-and-datagrid)
  - [Type Setup](#type-setup)
  - [Global Script Setup](#global-script-setup)
  - [Page-Script Setup](#page-script-setup)
  - [Page Setup](#page-setup)
  - [Page.Load Event Setup](#pageload-event-setup)
  - [Button.Click Event Setup](#buttonclick-event-setup)
- [Styling](#styling)
  - [Applying the CSS](#applying-the-css)
  - [Customising CSS](#customising-css)
  - [CSS Upgrading](#css-upgrading)

## Version 

1.1 DataGrid Editing
1. Added code to return error when ID column is not found
2. Added code to skip rules that do not have corresponding columns
3. Added code to detect uniqueness of DataGrid class on page

1.2 Updated script to cater for changed DataGrid rendering

1.3 Added custom event handler feature

1.4 Fixed Selectable column bug

1.5 Enabled adding ID column as number; fixed bug: no ID column in result

# Setup

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

## Global Script Setup
1. Create a Global Script called "EditableDataGrid"
2. Add four input parameters to the Global Script
   1. ButtonClassName
   2. DataGridClass
   3. FormFields
   4. IdentityColumnHeader
   5. CallbackScript
3. Drag a *JavaScript* action into the script
4. Add the Javascript below into the JavaScript code property
```javascript
/* Stadium Script Version 1.5 https://github.com/stadium-software/datagrid-inline-edit */
let scope = this;
let random =  Math.round(Math.random() * 1000);
resetDataGrid();
let callback = ~.Parameters.Input.CallbackScript;
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
let buttons = document.querySelectorAll("." + buttonParentClass);
if (buttons.length == 0) {
    console.error("There seems to be no button assigned to the DataGrid. Add a button to the page, assign a class to the button and add it to the 'ButtonClassName' parameter when you call the script");
    return false;
} else if (buttons.length > 1) {
    console.error("The class '" + buttonParentClass + "' is assigned to multiple buttons on this page. Buttons using this script must have unique classnames");
    return false;
}
let setRowEditingMode = (e) => {
    let currentRow = e.target.closest("tr");
    let editing = document.querySelector(".editing");
    if (editing) {
        editing.classList.remove("editing");
    }
    currentRow.classList.add("editing");
};
let getIndex = (heystack, needle) => {
    return heystack.findIndex((col) => col.name == needle);
};

insertForm();
initForm();

function initForm(){
    let formFields = enrichFormFields(rowFormFields);
    let result = formFields.find((obj) => {
        return obj.type == "Identity";
    });
    if (!result) {
        console.error("The identity column was not found");
        return false;
    }

    let rows = table.querySelectorAll("tbody tr");
    let clonedTable = table.cloneNode(true);
    clonedTable.id = 'table-clone';
    let clonedRows = clonedTable.querySelectorAll("tbody tr");
    for (let j = 0; j < rows.length; j++){
        let origRow = rows[j];
        let editRow = clonedRows[j];
        editRow.id = origRow.id + "-clone";
        editRow.setAttribute("identity", editRow.querySelector("td:nth-child(" + result.colNo + ")").textContent);
        for (let i = 0; i < formFields.length; i++) {
            let origCell = origRow.querySelectorAll("td")[i];
            let editCell = editRow.querySelectorAll("td")[i];
            let name = formFields[i].name;
            let value = origCell.textContent;
            let type = formFields[i].type;
            let data = formFields[i].data;
            let min = formFields[i].min;
            let max = formFields[i].max;
            let required = formFields[i].required;
            let el;
            editRow.addEventListener("click", setRowEditingMode);
            if (type == "date") {
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
            } else if (type == "text") {
                el = document.createElement("input");
                el.value = value;
                el.setAttribute("stadium-form-name", name);
                el.setAttribute("type", type);
                el.classList.add("form-control");
            } else {
                if (!origCell.querySelector("button")) {
                    el = document.createElement("input");
                    el.value = value;
                    el.setAttribute("type", "hidden");
                    el.setAttribute("stadium-form-name", name);
                    editCell.textContent = value;
                    el = document.createElement("span");
                    el.textContent = value;
                } else { 
                    el = document.createElement("span");
                }
            }
            el.classList.add("stadium-inline-form-control");
            if (required) el.setAttribute("required", "");
            editCell.innerHTML = "";
            editCell.appendChild(el);
        }
    }
    dg.appendChild(clonedTable);
    preparePage(clonedTable);
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

/*--------------------------------------------------------------------------------------*/

function preparePage(clonedTbl) { 
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
    buttonBar.classList.add("edit-button-bar");
    editButton.classList.add("visually-hidden");
    
    let saveButton = document.createElement("button");
    saveButton.setAttribute("class", "btn btn-lg btn-default");
    saveButton.setAttribute("type", "submit");
    saveButton.setAttribute("form", "form" + random);
    saveButton.innerText = "Save";

    let cancelLink = document.createElement("a");
    cancelLink.setAttribute("class", "btn btn-lg btn-link");
    cancelLink.innerText = "Cancel";
    cancelLink.addEventListener("click", resetDataGrid);
    
    buttonBar.appendChild(saveButton);
    buttonBar.appendChild(cancelLink);
    editButtonParent.appendChild(buttonBar);

    let arrHeadings = clonedTbl.querySelectorAll("thead th");
    for (let i = 0; i < arrHeadings.length; i++) {
        arrHeadings[i].childNodes[0].classList.add("visually-hidden");
        let heading = document.createElement("span");
        heading.innerText = arrHeadings[i].innerText;
        heading.classList.add("inline-edit-heading");
        arrHeadings[i].innerHTML = heading.innerHTML;
    }

    clonedTbl.querySelector("tfoot").remove();
    clonedTbl.classList.add("edit-table");
    table.classList.add("edit-orig-table");
}
async function saveButtonClick(e) { 
    e.preventDefault();
    let rows = document.querySelectorAll("#table-clone tbody tr");
    let arrGridData = [];
    for (let j = 0; j < rows.length; j++) {
        let formFields = rows[j].querySelectorAll("[stadium-form-name]");
        let arrData = [];
        let rowID = rows[j].getAttribute("Identity");
        arrData.push({ Name: "Identity", Value: rowID });
        for (let i = 0; i < formFields.length; i++) {
            let fieldValue = formFields[i].value;
            if (formFields[i].getAttribute("type") == "checkbox") fieldValue = formFields[i].checked;
            let field = { Name: formFields[i].getAttribute("stadium-form-name"), Value: fieldValue };
            arrData.push(field);
        }
        arrGridData.push(arrData);
    }
    await scope[callback](arrGridData);
    resetDataGrid();
}
function enrichFormFields(data) {
    let arrHeadings = table.querySelectorAll("thead th");
    data.forEach((element, index) => {
        if (element.name) {
            data[index].name = element.name.toLowerCase().replaceAll(" ", "");
        } else { 
            element.name = "";
        }
    });
    for (let i = 0; i < arrHeadings.length; i++) {
        let heading = arrHeadings[i].innerText.toLowerCase().replaceAll(" ", "");
        let index = getIndex(data, heading);
        if (index > -1) {
            data[index].colNo = i + 1;
        } else if (IDColumn.toString().toLowerCase() == arrHeadings[i].innerText.toLowerCase()) {
            data.push({ name: IDColumn, colNo: i + 1, type: "Identity" });
        } else if (IDColumn == (i + 1)) {
            data.push({ name: "Identity", colNo: IDColumn, type: "Identity" });
        } else { 
            data.push({colNo: i + 1, name: heading});
        }
    }
    data.sort((a, b) => a.colNo - b.colNo);
    return data;
}
function resetDataGrid() { 
    let editTable = document.getElementById("table-clone");
    if (editTable) { 
        editTable.remove();
        let form = document.getElementById("form" + random);
        form.parentElement.insertBefore(dg, form);
        form.remove();
        document.querySelector(".edit-orig-table").classList.remove("edit-orig-table");
        document.querySelector(".stadium-inline-edit-datagrid-button button").classList.remove("visually-hidden");
        document.querySelector(".stadium-inline-edit-datagrid-button").classList.remove("stadium-inline-edit-datagrid-button");
        document.querySelector(".stadium-inline-edit-datagrid").classList.remove("stadium-inline-edit-datagrid");
        document.querySelector(".edit-button-bar").remove();
    }
}
function insertForm() { 
    let form = document.createElement('form');
    form.id = "form" + random;
    form.classList.add("datagrid-inline-edit-form");
    form.addEventListener("submit", saveButtonClick);
    dg.parentElement.insertBefore(form, dg);
    form.appendChild(dg);
}
```

## Page-Script Setup
1. Create a Script inside of the Page with any name you like (e.g. "SaveGrid")
2. Add one input parameter to the Script
   1. GridData
3. Drag a *Notification* action into the script
4. In the *Message* property, select the *GridData* parameter from the *Script Input Parameters* category

## Page Setup
1. Drag a *Button* control into the page and enter some text (e.g. Edit DataGrid)
2. Add a class of your choosing to the *Button* *Classes* property to uniquely identify this button (e.g. datagrid-inline-edit-button)
3. Drag a *DataGrid* control to the page ([see above](#database-connector-and-datagrid))
4. Add a class of your choosing to the *DataGrid* *Classes* property (e.g datagrid-inline-edit)
5. Note: If multiple editable DataGrids are shown on one page, each DataGrid and each corresponding edit button must have unique classnames

## Page.Load Event Setup
1. Populate the DataGrid with data ([see above](#database-connector-and-datagrid))

## Button.Click Event Setup
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
   4. IdentityColumnHeader: The header of the column that uniquely identifies each row (e.g. ID) OR the column number (e.g. 1)
   5. CallbackScript: The name of the page-level script that will process the updated data (e.g. SaveGrid)

![Inline Editing Input Parameters](images/InlineEditingInputParameters.png)

## Applying the CSS
The CSS below is required for the correct functioning of the module. Some elements can be [customised](#customising-css) using a variables CSS file. 

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

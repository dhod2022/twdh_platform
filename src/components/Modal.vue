<template>
  <div class="modal-backdrop" style="display: flex; flex-direction: column; height: 100vh;">
    <div class="modal">
      <header class="modal-header">
        <slot name="header">
          <h1>匯出Metadata</h1>
        </slot>
        <button
          type="button"
          class="btn-close"
          @click="close"
        >x</button>
      </header>

      <section class="modal-body">
        <slot name="body">
	<div class="body_div">
		第一步，產生新的DocuXML<br>
	  	DocuSky Corpus名稱: <input v-model="corpus_name" style="margin-left:9px; border: 1px solid #000000;">
	  	<button @click="Convert2DocuXML()" style="margin-left:20px;">按此產生DocuXML</button>
	  	<button @click="DownloadDocuXML" style="margin-left:20px;">下載DocuXML</button><br>
	  	第二步，上傳DocuXML至DocuSky<br>
	  	DocuSky 資料庫名稱: <input v-model="database_name" style="margin-left:10px; border: 1px solid #000000;">
	  	<button @click="UploadDocuXML2DocuSky()" style="margin-left:20px;">上傳DocuSky</button><br>
  		
	</div>
	  <div class="body_div">
	  	<br><h2>DocuXML預覽</h2>
	  	<pre style="height: 100vh; overflow-y: auto;">{{ DocuXML }}</pre>
	  </div>
        </slot>
       </section>

      <footer class="modal-footer">
        <slot name="footer">
        </slot>
        <button
          type="button"
          class="btn-green"
          @click="close"
        >
          返回原來頁面
        </button>
      </footer>
    </div>
  </div>
</template>

<script>
import axios from 'axios';
import tsainiancheng_mission5 from '../modules/opendata/js/tsai.js';

const default_mapping = {'Metadata_ID': 'filename', '來源系統': 'doc_source', '來源系統縮寫': 'metadata/doc_source', '文件原系統頁面URL': 'metadata/doc_source_href',
	'題名': 'title', '摘要': 'doc_content', '類目階層': 'metadata/category_level', '西元年': 'year_for_grouping', '起始時間': 'timeseq_not_before', '結束時間': 'timeseq_not_after',
	'典藏號': 'metadata/collection_number',
	'相關人員': 'metatags/PersonName', '相關地點': 'metatags/PlaceName', '相關組織': 'metatags/Organization', '關鍵詞': 'metatags/Keywords'}; // [original]另外處理
	
const default_config = {
	'db_name': 'default',
	'corpus': 'Mycorpus',
	'DocuXML_filename': 'DocuXML',
	'mapping': default_mapping,
};

var filename_col, doc_content_col;
var metadata_cols, metatags_cols, original_cols;
var col_arr;

  export default {
    name: 'modal',
    props: { Metadata: '', username: "" },
    data:  () => {
	return {
	  database_name: "default",
	  corpus_name: "corpus",
	  DocuXML: "",
	 }
	},
    methods: {
    	Convert2DocuXML() {
	  	var data = this.Metadata;
	  	var config = {
			'db_name': this.database_name,
			'corpus': this.corpus_name,
		};
	  	
		for (let i in data)
			for (const [key, value] of Object.entries(data[i]))
				data[i][key] = (data[i][key]===null)? '':data[i][key].toString();
		config = Object.assign({}, default_config, config);
		config['DocuXML_filename'] = config['DocuXML_filename'] + '.xml';
		var DocuXML = tsainiancheng_mission5.processing(data, config);
		this.DocuXML = DocuXML["file"]["value"];
    	},
      	UploadDocuXML2DocuSky() {
      		SessionKey = "DocuSky_SID=" + $cookies.get("DocuSky_SID");
		docuskyManageDbListSimpleUI.uploadMultipart(this.DocuXML, function(data) {alert("DocuSky 上傳成功");}, function() {alert("DocuSky 上傳失敗");});
		
		  var formData = [];
		  var UserId = 10489;
		  /*$.post("../../../OD/webApi/getUserIdFromUserName.php", {"username": this.username}, function(data){
		    console.log(data.message);
		    UserId = data.message;
		  });*/
		  
		  formData[0] = {};
		  formData[0]["name"] = "dbTitleForImport";
		  formData[0]["value"] = "dbTitle";
		  formData[1] = {};
		  formData[1]["name"] = "type";
		  formData[1]["value"] = "XML";
		  formData[2] = {};
		  formData[2]["name"] = "userId";
		  formData[2]["value"] = UserId;
		  
		formData.file = Object.assign({}, this.DocuXML.file);
		formData.file["name"] = "importedFiles";
		odManageDbListSimpleUI.uploadTempFiles(formData, function(data) {alert("OD 上傳成功");}, function() {alert("OD 上傳失敗");});
		
		//axios.post("https://skolem.csie.ntu.edu.tw/OD/tsainiancheng/saveDocuXML.php", {DocuXML: this.DocuXML, filename: "DocuXML.xml"}).then(function(response) {});
		
	},
	DownloadDocuXML() {
		function downloadURI(uri, name) {
		  var link = document.createElement("a");
		  link.download = name;
		  link.href = uri;
		  document.body.appendChild(link);
		  link.click();
		  document.body.removeChild(link);
		}
		downloadURI("data:text/html,"+encodeURI(this.DocuXML), "DocuXML.xml");
	},
      close() {
        this.$emit('close');
      },
    },
    mounted() {
      let docuScript = document.createElement('script');
      docuScript.setAttribute('src', 'https://skolem.csie.ntu.edu.tw/DocuSky/js.ui/docusky.ui.manageDbListSimpleUI.js');
      let ODScript = document.createElement('script');
      ODScript.setAttribute('src', 'https://skolem.csie.ntu.edu.tw/OD/js.ui/od.ui.manageDbListSimpleUI.js');
      
      // prevent double loading script
	let jqueryflag = 1, docuWidgetsflag = 1, ODWidgetsflag = 1;
	for (let node of document.head.childNodes) {
		if (node["src"] == 'https://skolem.csie.ntu.edu.tw/DocuSky/js.ui/docusky.ui.manageDbListSimpleUI.js')
			docuWidgetsflag = 0;
		if (node["src"] == 'https://skolem.csie.ntu.edu.tw/OD/js.ui/od.ui.manageDbListSimpleUI.js')
			ODWidgetsflag = 0;
		}
	if (docuWidgetsflag == 1)		
	      document.head.appendChild(docuScript);
	if (ODWidgetsflag == 1)
		document.head.appendChild(ODScript);
    }
  };
</script>

<style>
  .modal-backdrop {
    position: fixed;
    top: 0;
    bottom: 0;
    left: 0;
    right: 0;
    background-color: rgba(0, 0, 0, 0.3);
    display: flex;
    justify-content: center;
    align-items: center;
  }

  .modal {
    background: #FFFFFF;
    box-shadow: 2px 2px 20px 1px;
    display: flex;
    flex-direction: column;
  }

  .modal-header,
  .modal-footer {
    padding: 15px;
    display: flex;
  }

  .modal-header {
    position: relative;
    border-bottom: 1px solid #eeeeee;
    color: #4AAE9B;
    justify-content: space-between;
  }

  .modal-footer {
    border-top: 1px solid #eeeeee;
    flex-direction: column;
    justify-content: flex-end;
  }

  .modal-body {
    position: relative;
    padding: 20px 10px;
    overflow-y: hidden;
  }

  .btn-close {
    position: absolute;
    top: 0;
    right: 0;
    border: none;
    font-size: 20px;
    padding: 10px;
    cursor: pointer;
    font-weight: bold;
    color: #4AAE9B;
    background: transparent;
  }

  .btn-green {
    color: white;
    background: #4AAE9B;
    border: 1px solid #4AAE9B;
    border-radius: 2px;
  }
</style>


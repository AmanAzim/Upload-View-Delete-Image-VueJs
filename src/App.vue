<template>
  <div class="App" id="app">
    <div class="container-fluid mt-5">

      <div class="row p-5">
        <div class="offset-md-3 col-md-6">
          <form>
            <div class="input-group text-center">
              <input type="file" multiple class="form-control" v-on:change="fileSelectHandler"/>
            </div>
          </form>
        </div>
        <div class="offset-md-3 col-md-6 mt-3">
          <button class="btn btn-primary"  v-bind:disabled="uploadStart || images.length<=0" v-on:click="fileUploadHandler">
            {{uploadStart? 'Delete all items to upload again':'Upload Image'}}
          </button>
        </div>
        <div class="offset-md-3 col-md-6 mt-3">
          <div class="progress">
            <div class="progress-bar"  :style="{width:progress+'%'}">{{progress}}</div>
          </div>
        </div>
      </div>
      <hr>
      <div class="row mt-5">
        <h5 class="mb-2 text-center col-sm-12">Your selected images:</h5>
      </div>
      <div class="row mt-2 mb-2">
        <div class="offset-md-3 col-md-7 mt-3 text-center">
          <span  class="mx-2 my-5" v-for="file in images" :key="file.id">
            <img :src="file.img" width="100px" height="100px"  />
            <button class="btn btn-danger" v-on:click="deleteFileHandler(file.id)">Delete</button>
          </span>
        </div>
      </div>
      <hr>
      <div class="row mt-2">
        <h5 class="offset-md-3 col-md-6 mt-3">Your uploaded images:</h5>
        <div class="offset-md-2 col-md-8 mt-3">
          <img  v-for="img in uploadedImagesUrl" v-bind:src="img.downloadURL" :key="img.id" width="100px" height="100px" style="margin:20px"/>
        </div>
      </div>
      <hr>
    </div>
  </div>
</template>

<script>
import {storageRef} from './firebase/index';
import 'bootstrap/dist/css/bootstrap.css';

export default {
  data(){
    return {
      filesToUpload:[],
      images:[],
      uploadedImagesUrl:[],
      progress:0,
      uploaded:false,
      uploadStart:false
    }
  },
  methods:{
    fileSelectHandler(event){
      event.preventDefault();

      const imageFiles = event.target.files; //document.getElementById("image"); //It gives all the uploaded images
      const filesLength = imageFiles.length; // imageFiles.files.length;

      for(var i = 0; i < filesLength; i++) {

        let imgId=Math.random();

        //processing the raw image for viewing withing component
        let reader = new FileReader();
        let file = imageFiles[i];
        reader.onloadend = () => {
          this.images=[...this.images, {id:imgId, img:reader.result}]
        };
        reader.readAsDataURL(file);


        //Storing the raw image for upload
        let imgName=imageFiles[i].name;
        this.filesToUpload=[...this.filesToUpload, {id:imgId, img:file, name:imgName}];
      }
    },
    fileUploadHandler(event) {
      event.preventDefault();

      let P = new Promise((resolve) => {
          this.uploadedImagesUrl = []
          this.uploadStart = true;
          if(this.uploadedImagesUrl.length<=0 && this.uploadStart){
              resolve();
          }
      });
      //Inside then block that executes after resolve changes the state
      P.then(() => {
        this.filesToUpload.forEach((file)=>{

          //Storing the image to firebase under "my_images" folder
          const uploadTask = storageRef.child('my_images_vue/' + file.name).put(file.img);

          //Three call back functions upon each upload operation 1.indicates progress, 2.shows error, 3. if successful gives the uploaded images URL
          uploadTask.on('state_changed', (snapshot) => {

            /*indicates Progress*/
            let uploadProgress = Math.round((snapshot.bytesTransferred / snapshot.totalBytes) * 100);
            this.progress = this.progress + uploadProgress;


          }, (error)=>{
            console.log(error);
          }, ()=>{

            /*Indicates task Completation*/
            //If successful then get the url of the uploaded image
            uploadTask.snapshot.ref.getDownloadURL().then((downloadURL) => {
              console.log('File available at', downloadURL);
              this.uploadedImagesUrl = [...this.uploadedImagesUrl, {id: file.id, downloadURL: downloadURL}];
              this.progress = 0;

            })

          });
          this.uploaded = true;//Indicates the file is uploaded
        });//Inside then block that executes after resolve changes the state
      });
    },
    deleteFileHandler(id){
      //removing the image from the "images" array that we use for viewing the image
      let tempImages=[...this.images];
      tempImages=tempImages.filter(img=>img.id!==id);
      this.images=tempImages;


      let tempfilesToUpload=[...this.filesToUpload];


      //Getting the exact file to delete from the array of raw images from where we uploaded the images
      const file=tempfilesToUpload.find(file=>file.id===id);

      //Deleting the raw image from the array for uploading images(already been uploaded)
      tempfilesToUpload=tempfilesToUpload.filter(file=>file.id!==id);
      this.filesToUpload=tempfilesToUpload;

      //Safety check so that we only try to delete from firebase if the image is uploaded
      if(this.uploaded){
        var desertRef = storageRef.child('my_images_vue/'+file.img.name);//// Create a reference to the file to delete
        // Delete the file
        desertRef.delete().then(()=>{

          //After deleting removing the URL of the deleted image fro the list of uploaded images URL so it that it cannot be rendered
          let tempUploadedImagesUrl=[...this.uploadedImagesUrl];
          tempUploadedImagesUrl=tempUploadedImagesUrl.filter(item=>item.id!==id);

          return new Promise(resolve =>{
            this.uploadedImagesUrl=tempUploadedImagesUrl;
            resolve()
          }).then(()=>{
            if(this.uploadedImagesUrl.length<=0){
              this.uploadStart=false;
            }
          })

        }).catch((error)=>console.log(error));
      }
    },
  }
}
</script>

<style>
  #app {
    font-family: 'Avenir', Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    text-align: center;
    color: #2c3e50;
    margin-top: 60px;
  }
</style>

buildscript {
    repositories { google(); mavenCentral() }
    dependencies { classpath "com.android.tools.build:gradle:8.1.0" }
}
allprojects { repositories { google(); mavenCentral() } }
com.android.tools.build


plugins {
    id 'com.android.application'
    id 'org.jetbrains.kotlin.android'
}

android {
    namespace "com.example.aiimageapp"
    compileSdk 34

    defaultConfig {
        applicationId "com.example.aiimageapp"
        minSdk 24
        targetSdk 34
        versionCode 1
        versionName "1.0"
    }

    compileOptions { sourceCompatibility JavaVersion.VERSION_17; targetCompatibility JavaVersion.VERSION_17 }
    kotlinOptions { jvmTarget = "17" }
}

dependencies {
    implementation "org.jetbrains.kotlin:kotlin-stdlib:1.9.0"
    implementation 'androidx.core:core-ktx:1.10.1'
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'com.google.android.material:material:1.9.0'
    implementation 'androidx.constraintlayout:constraintlayout:2.1.4'

    // Networking + coroutines
    implementation 'com.squareup.retrofit2:retrofit:2.9.0'
    implementation 'com.squareup.retrofit2:converter-gson:2.9.0'
    implementation "org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3"
    implementation "com.squareup.okhttp3:okhttp:4.11.0"
    implementation "com.squareup.okhttp3:logging-interceptor:4.11.0"

    // Image loading
    implementation 'com.github.bumptech.glide:glide:4.15.1'
    kapt 'com.github.bumptech.glide:compiler:4.15.1'

    // Activity result APIs
    implementation "androidx.activity:activity-ktx:1.8.0"
}

<manifest package="com.example.aiimageapp" xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.INTERNET"/>
    <uses-permission android:name="android.permission.CAMERA"/>
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>

    <application
        android:allowBackup="true"
        android:label="AI Image App"
        android:theme="@style/Theme.AppCompat.Light.NoActionBar">
        <activity android:name=".MainActivity" android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
    </application>
</manifest>

android.permission.CAMERA

<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent" android:layout_height="match_parent"
    android:padding="16dp">

    <ImageView
        android:id="@+id/imageView"
        android:layout_width="0dp"
        android:layout_height="0dp"
        android:scaleType="centerCrop"
        android:contentDescription="selected image"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintBottom_toTopOf="@+id/resultImage"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent"/>

    <ImageView
        android:id="@+id/resultImage"
        android:layout_width="0dp"
        android:layout_height="200dp"
        android:scaleType="centerCrop"
        android:contentDescription="ai result"
        app:layout_constraintTop_toBottomOf="@id/imageView"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintBottom_toTopOf="@+id/buttonsLayout"
        android:layout_marginTop="8dp"/>

    <LinearLayout
        android:id="@+id/buttonsLayout"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:gravity="center"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent">

        <Button
            android:id="@+id/btnPick"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="اختيار صورة"/>

        <Button
            android:id="@+id/btnCamera"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="كاميرا"
            android:layout_marginStart="8dp"/>

        <Button
            android:id="@+id/btnSend"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="أرسل للذكاء الاصطناعي"
            android:layout_marginStart="8dp"/>
    </LinearLayout>
</androidx.constraintlayout.widget.ConstraintLayout>

package com.example.aiimageapp

import android.Manifest
import android.content.ContentValues
import android.content.pm.PackageManager
import android.graphics.Bitmap
import android.net.Uri
import android.os.Build
import android.os.Bundle
import android.provider.MediaStore
import android.widget.Button
import android.widget.ImageView
import android.widget.Toast
import androidx.activity.result.contract.ActivityResultContracts
import androidx.appcompat.app.AppCompatActivity
import androidx.core.content.ContextCompat
import com.bumptech.glide.Glide
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext
import okhttp3.MediaType.Companion.toMediaTypeOrNull
import okhttp3.MultipartBody
import okhttp3.OkHttpClient
import okhttp3.RequestBody
import okhttp3.RequestBody.Companion.asRequestBody
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory
import java.io.File
import java.io.FileOutputStream

class MainActivity : AppCompatActivity() {

    private lateinit var imageView: ImageView
    private lateinit var resultImage: ImageView
    private var currentImageUri: Uri? = null

    private val pickImage = registerForActivityResult(ActivityResultContracts.GetContent()) { uri: Uri? ->
        uri?.let {
            currentImageUri = it
            Glide.with(this).load(it).into(imageView)
        }
    }

    private val takePhoto = registerForActivityResult(ActivityResultContracts.TakePicture()) { success ->
        if (success && currentImageUri != null) {
            Glide.with(this).load(currentImageUri).into(imageView)
        } else {
            Toast.makeText(this, "فشل التقاط الصورة", Toast.LENGTH_SHORT).show()
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        imageView = findViewById(R.id.imageView)
        resultImage = findViewById(R.id.resultImage)
        val btnPick: Button = findViewById(R.id.btnPick)
        val btnCamera: Button = findViewById(R.id.btnCamera)
        val btnSend: Button = findViewById(R.id.btnSend)

        btnPick.setOnClickListener { pickImage.launch("image/*") }

        btnCamera.setOnClickListener {
            if (checkPermission(Manifest.permission.CAMERA)) {
                val uri = createImageUri() // create a place to save
                currentImageUri = uri
                takePhoto.launch(uri)
            } else {
                requestPermissions(arrayOf(Manifest.permission.CAMERA), 1001)
            }
        }

        btnSend.setOnClickListener {
            currentImageUri?.let { uri ->
                // convert to file and send
                CoroutineScope(Dispatchers.Main).launch {
                    val f = uriToFile(uri)
                    if (f != null) {
                        sendImageToAi(f)
                    } else {
                        Toast.makeText(this@MainActivity, "خطأ في قراءة الملف", Toast.LENGTH_SHORT).show()
                    }
                }
            } ?: Toast.makeText(this, "اختَر صورة أولاً", Toast.LENGTH_SHORT).show()
        }
    }

    private fun checkPermission(permission: String): Boolean {
        return ContextCompat.checkSelfPermission(this, permission) == PackageManager.PERMISSION_GRANTED
    }

    private fun createImageUri(): Uri? {
        val resolver = contentResolver
        val contentValues = ContentValues().apply {
            put(MediaStore.MediaColumns.DISPLAY_NAME, "temp_ai_${System.currentTimeMillis()}.jpg")
            put(MediaStore.MediaColumns.MIME_TYPE, "image/jpeg")
        }
        return resolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, contentValues)
    }

    private suspend fun uriToFile(uri: Uri): File? = withContext(Dispatchers.IO) {
        try {
            val input = contentResolver.openInputStream(uri) ?: return@withContext null
            val file = File(cacheDir, "upload_${System.currentTimeMillis()}.jpg")
            FileOutputStream(file).use { out ->
                input.copyTo(out)
            }
            file
        } catch (e: Exception) {
            e.printStackTrace()
            null
        }
    }

    private fun sendImageToAi(file: File) {
        // build retrofit and call
        val client = OkHttpClient.Builder().build()
        val retrofit = Retrofit.Builder()
            .baseUrl("https://your-ai-endpoint.example/") // <-- استبدلها بنقطة النهاية الحقيقية
            .client(client)
            .addConverterFactory(GsonConverterFactory.create())
            .build()

        val service = retrofit.create(ApiService::class.java)

        val reqFile = file.asRequestBody("image/jpeg".toMediaTypeOrNull())
        val body = MultipartBody.Part.createFormData("image", file.name, reqFile)

        CoroutineScope(Dispatchers.IO).launch {
            try {
                val resp = service.uploadImage("Bearer YOUR_API_KEY", body) // استبدل رأس التفويض
                if (resp.isSuccessful && resp.body() != null) {
                    val bodyResp = resp.body()!!
                    // نتوقع JSON يحوي "result_image_url" أو "result_base64"
                    val imageUrl = bodyResp.result_image_url
                    withContext(Dispatchers.Main) {
                        if (!imageUrl.isNullOrEmpty()) {
                            Glide.with(this@MainActivity).load(imageUrl).into(resultImage)
                        } else if (!bodyResp.result_base64.isNullOrEmpty()) {
                            // لو رجع base64: نكتب كـ temp file ثم نعرض
                            val decoded = android.util.Base64.decode(bodyResp.result_base64, android.util.Base64.DEFAULT)
                            val outFile = File(cacheDir, "ai_result_${System.currentTimeMillis()}.jpg")
                            outFile.writeBytes(decoded)
                            Glide.with(this@MainActivity).load(outFile).into(resultImage)
                        } else {
                            Toast.makeText(this@MainActivity, "لم يصل نتيجة صالحة", Toast.LENGTH_SHORT).show()
                        }
                    }
                } else {
                    withContext(Dispatchers.Main) {
                        Toast.makeText(this@MainActivity, "خطأ في الخادم: ${resp.code()}", Toast.LENGTH_SHORT).show()
                    }
                }
            } catch (e: Exception) {
                e.printStackTrace()
                withContext(Dispatchers.Main) {
                    Toast.makeText(this@MainActivity, "خطأ إرسال: ${e.message}", Toast.LENGTH_LONG).show()
                }
            }
        }
    }

    // handle permission result simple
    override fun onRequestPermissionsResult(requestCode: Int, permissions: Array<out String>, grantResults: IntArray) {
        if (requestCode == 1001 && grantResults.isNotEmpty() && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
            val uri = createImageUri()
            currentImageUri = uri
            takePhoto.launch(uri)
        } else {
            Toast.makeText(this, "محتاج صلاحية الكاميرا", Toast.LENGTH_SHORT).show()
        }
        super.onRequestPermissionsResult(requestCode, permissions, grantResults)
    }
}
com.example.aiimageapp
android.Manifest
android.content.ContentValues
android.content.pm.PackageManager
android.graphics.Bitmap
android.net.Uri
android.os.Build
android.os.Bundle
android.provider.MediaStore
android.widget.Button
android.widget.ImageView
android.widget.Toast
androidx.activity.result.contract.ActivityResultContracts
androidx.appcompat.app.AppCompatActivity
androidx.core.content.ContextCompat
com.bumptech.glide.Glide
kotlinx.coroutines.CoroutineScope
kotlinx.coroutines.Dispatchers
kotlinx.coroutines.launch
kotlinx.coroutines.withContext
okhttp3.MediaType.Companion.toMediaTypeOrNull
okhttp3.MultipartBody
okhttp3.OkHttpClient
okhttp3.RequestBody
okhttp3.RequestBody.Companion.asRequestBody
retrofit2.Retrofit
retrofit2.converter.gson.GsonConverterFactory
java.io.File
java.io.FileOutputStream
btnPick.setOnClickListener
btnCamera.setOnClickListener
Manifest.permission.CAMERA
btnSend.setOnClickListener
Dispatchers.IO
file.name
1001


package com.example.aiimageapp

import okhttp3.MultipartBody
import retrofit2.Response
import retrofit2.http.*

data class AiResponse(
    val result_image_url: String?,   // إذا كانت الخدمة تعيد رابط
    val result_base64: String?       // أو صورة بصيغة base64
)

interface ApiService {
    @Multipart
    @POST("v1/process-image") // ضع المسار الصحيح من مزودك
    suspend fun uploadImage(
        @Header("Authorization") auth: String,
        @Part image: MultipartBody.Part
    ): Response<AiResponse>
}
com.example.aiimageappokhttp3.MultipartBody
retrofit2.Response
MultipartBody.Part


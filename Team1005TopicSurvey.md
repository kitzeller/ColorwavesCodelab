#Topic Survey: Voice Recognition and Colorwaves

###### Team 1005: Jacob, Madeline & Kit


## Overview of Tutorial

We're going to make an app called "Colorwaves". It well let you control the color of your screen purely from your voice.

Prerequisites:

* Android Studio 3.2
* Android physical or virtual devices API 27
* Google Cloud


## Starter Code

If you do not have the starter code android files, please follow the steps to make the base application. If you have the code, import it as a project in Android Studio and feel free to move to the next section.

1. Make a new project called Colorwaves
2. Select Basic Activity
3. Make a class called **GoogleCredentialsInterceptor** using the code below:


```java
import com.google.auth.Credentials;

import java.io.IOException;
import java.net.URI;
import java.net.URISyntaxException;
import java.util.List;
import java.util.Map;

import io.grpc.CallOptions;
import io.grpc.Channel;
import io.grpc.ClientCall;
import io.grpc.ClientInterceptor;
import io.grpc.ClientInterceptors;
import io.grpc.Metadata;
import io.grpc.MethodDescriptor;
import io.grpc.Status;
import io.grpc.StatusException;

/**
 * Created by brijesh on 22/7/17.
 */

public class GoogleCredentialsInterceptor implements ClientInterceptor {

    private final Credentials mCredentials;

    private Metadata mCached;

    private Map<String, List<String>> mLastMetadata;

    GoogleCredentialsInterceptor(Credentials credentials) {
        mCredentials = credentials;
    }

    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(final MethodDescriptor<ReqT, RespT> method, CallOptions callOptions, final Channel next) {
        return new ClientInterceptors.CheckedForwardingClientCall<ReqT, RespT>(
                next.newCall(method, callOptions)) {
            @Override
            protected void checkedStart(Listener<RespT> responseListener, Metadata headers)
                    throws StatusException {
                Metadata cachedSaved;
                URI uri = serviceUri(next, method);
                synchronized (this) {
                    Map<String, List<String>> latestMetadata = getRequestMetadata(uri);
                    if (mLastMetadata == null || mLastMetadata != latestMetadata) {
                        mLastMetadata = latestMetadata;
                        mCached = toHeaders(mLastMetadata);
                    }
                    cachedSaved = mCached;
                }
                headers.merge(cachedSaved);
                delegate().start(responseListener, headers);
            }
        };
    }

    /**
     * Generate a JWT-specific service URI. The URI is simply an identifier with enough
     * information for a service to know that the JWT was intended for it. The URI will
     * commonly be verified with a simple string equality check.
     */
    private URI serviceUri(Channel channel, MethodDescriptor<?, ?> method)
            throws StatusException {
        String authority = channel.authority();
        if (authority == null) {
            throw Status.UNAUTHENTICATED
                    .withDescription("Channel has no authority")
                    .asException();
        }
        // Always use HTTPS, by definition.
        final String scheme = "https";
        final int defaultPort = 443;
        String path = "/" + MethodDescriptor.extractFullServiceName(method.getFullMethodName());
        URI uri;
        try {
            uri = new URI(scheme, authority, path, null, null);
        } catch (URISyntaxException e) {
            throw Status.UNAUTHENTICATED
                    .withDescription("Unable to construct service URI for auth")
                    .withCause(e).asException();
        }
        // The default port must not be present. Alternative ports should be present.
        if (uri.getPort() == defaultPort) {
            uri = removePort(uri);
        }
        return uri;
    }

    private URI removePort(URI uri) throws StatusException {
        try {
            return new URI(uri.getScheme(), uri.getUserInfo(), uri.getHost(), -1 /* port */,
                    uri.getPath(), uri.getQuery(), uri.getFragment());
        } catch (URISyntaxException e) {
            throw Status.UNAUTHENTICATED
                    .withDescription("Unable to construct service URI after removing port")
                    .withCause(e).asException();
        }
    }

    private Map<String, List<String>> getRequestMetadata(URI uri) throws StatusException {
        try {
            return mCredentials.getRequestMetadata(uri);
        } catch (IOException e) {
            throw Status.UNAUTHENTICATED.withCause(e).asException();
        }
    }

    private static Metadata toHeaders(Map<String, List<String>> metadata) {
        Metadata headers = new Metadata();
        if (metadata != null) {
            for (String key : metadata.keySet()) {
                Metadata.Key<String> headerKey = Metadata.Key.of(
                        key, Metadata.ASCII_STRING_MARSHALLER);
                for (String value : metadata.get(key)) {
                    headers.put(headerKey, value);
                }
            }
        }
        return headers;
    }

}
```

4.Make a class called **VoiceRecorder**

```java
/*
 * Copyright 2016 Google Inc. All Rights Reserved.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package com.app.androidkt.speechapi;

import android.media.AudioFormat;
import android.media.AudioRecord;
import android.media.MediaRecorder;
import android.support.annotation.NonNull;


/**
 * Continuously records audio and notifies the {@link VoiceRecorder.Callback} when voice (or any
 * sound) is heard.
 *
 * <p>The recorded audio format is always {@link AudioFormat#ENCODING_PCM_16BIT} and
 * {@link AudioFormat#CHANNEL_IN_MONO}. This class will automatically pick the right sample rate
 * for the device. Use {@link #getSampleRate()} to get the selected value.</p>
 */
public class VoiceRecorder {

    private static final int[] SAMPLE_RATE_CANDIDATES = new int[]{16000, 11025, 22050, 44100};

    private static final int CHANNEL = AudioFormat.CHANNEL_IN_MONO;
    private static final int ENCODING = AudioFormat.ENCODING_PCM_16BIT;

    private static final int AMPLITUDE_THRESHOLD = 1500;

    private static final int SPEECH_TIMEOUT_MILLIS = 2000;
    private static final int MAX_SPEECH_LENGTH_MILLIS = 30 * 1000;

    public static abstract class Callback {

        /**
         * Called when the recorder starts hearing voice.
         */
        public void onVoiceStart() {
        }

        /**
         * Called when the recorder is hearing voice.
         *
         * @param data The audio data in {@link AudioFormat#ENCODING_PCM_16BIT}.
         * @param size The size of the actual data in {@code data}.
         */
        public void onVoice(byte[] data, int size) {
        }

        /**
         * Called when the recorder stops hearing voice.
         */
        public void onVoiceEnd() {
        }
    }

    private final Callback mCallback;

    private AudioRecord mAudioRecord;

    private Thread mThread;

    private byte[] mBuffer;

    private final Object mLock = new Object();

    /** The timestamp of the last time that voice is heard. */
    private long mLastVoiceHeardMillis = Long.MAX_VALUE;

    /** The timestamp when the current voice is started. */
    private long mVoiceStartedMillis;

    public VoiceRecorder(@NonNull Callback callback) {
        mCallback = callback;
    }

    /**
     * Starts recording audio.
     *
     * <p>The caller is responsible for calling {@link #stop()} later.</p>
     */
    public void start() {
        // Stop recording if it is currently ongoing.
        stop();
        // Try to create a new recording session.
        mAudioRecord = createAudioRecord();
        if (mAudioRecord == null) {
            throw new RuntimeException("Cannot instantiate VoiceRecorder");
        }
        // Start recording.
        mAudioRecord.startRecording();
        // Start processing the captured audio.
        mThread = new Thread(new ProcessVoice());
        mThread.start();
    }

    /**
     * Stops recording audio.
     */
    public void stop() {
        synchronized (mLock) {
            dismiss();
            if (mThread != null) {
                mThread.interrupt();
                mThread = null;
            }
            if (mAudioRecord != null) {
                mAudioRecord.stop();
                mAudioRecord.release();
                mAudioRecord = null;
            }
            mBuffer = null;
        }
    }

    /**
     * Dismisses the currently ongoing utterance.
     */
    public void dismiss() {
        if (mLastVoiceHeardMillis != Long.MAX_VALUE) {
            mLastVoiceHeardMillis = Long.MAX_VALUE;
            mCallback.onVoiceEnd();
        }
    }

    /**
     * Retrieves the sample rate currently used to record audio.
     *
     * @return The sample rate of recorded audio.
     */
    public int getSampleRate() {
        if (mAudioRecord != null) {
            return mAudioRecord.getSampleRate();
        }
        return 0;
    }

    /**
     * Creates a new {@link AudioRecord}.
     *
     * @return A newly created {@link AudioRecord}, or null if it cannot be created (missing
     * permissions?).
     */
    private AudioRecord createAudioRecord() {
        for (int sampleRate : SAMPLE_RATE_CANDIDATES) {
            final int sizeInBytes = AudioRecord.getMinBufferSize(sampleRate, CHANNEL, ENCODING);
            if (sizeInBytes == AudioRecord.ERROR_BAD_VALUE) {
                continue;
            }
            final AudioRecord audioRecord = new AudioRecord(MediaRecorder.AudioSource.MIC,
                    sampleRate, CHANNEL, ENCODING, sizeInBytes);
            if (audioRecord.getState() == AudioRecord.STATE_INITIALIZED) {
                mBuffer = new byte[sizeInBytes];
                return audioRecord;
            } else {
                audioRecord.release();
            }
        }
        return null;
    }

    /**
     * Continuously processes the captured audio and notifies {@link #mCallback} of corresponding
     * events.
     */
    private class ProcessVoice implements Runnable {

        @Override
        public void run() {
            while (true) {
                synchronized (mLock) {
                    if (Thread.currentThread().isInterrupted()) {
                        break;
                    }
                    final int size = mAudioRecord.read(mBuffer, 0, mBuffer.length);
                    final long now = System.currentTimeMillis();
                    if (isHearingVoice(mBuffer, size)) {
                        if (mLastVoiceHeardMillis == Long.MAX_VALUE) {
                            mVoiceStartedMillis = now;
                            mCallback.onVoiceStart();
                        }
                        mCallback.onVoice(mBuffer, size);
                        mLastVoiceHeardMillis = now;
                        if (now - mVoiceStartedMillis > MAX_SPEECH_LENGTH_MILLIS) {
                            end();
                        }
                    } else if (mLastVoiceHeardMillis != Long.MAX_VALUE) {
                        mCallback.onVoice(mBuffer, size);
                        if (now - mLastVoiceHeardMillis > SPEECH_TIMEOUT_MILLIS) {
                            end();
                        }
                    }
                }
            }
        }

        private void end() {
            mLastVoiceHeardMillis = Long.MAX_VALUE;
            mCallback.onVoiceEnd();
        }

        private boolean isHearingVoice(byte[] buffer, int size) {
            for (int i = 0; i < size - 1; i += 2) {
                // The buffer has LINEAR16 in little endian.
                int s = buffer[i + 1];
                if (s < 0) s *= -1;
                s <<= 8;
                s += Math.abs(buffer[i]);
                if (s > AMPLITUDE_THRESHOLD) {
                    return true;
                }
            }
            return false;
        }
    }
}
```

5.Make a class called **SpeechAPI**. The code can be found below.

```java
import android.content.Context;
import android.content.SharedPreferences;
import android.os.AsyncTask;
import android.os.Handler;
import android.support.annotation.NonNull;
import android.util.Log;

import com.google.auth.oauth2.AccessToken;
import com.google.auth.oauth2.GoogleCredentials;
import com.google.cloud.speech.v1.RecognitionConfig;
import com.google.cloud.speech.v1.SpeechGrpc;
import com.google.cloud.speech.v1.SpeechRecognitionAlternative;
import com.google.cloud.speech.v1.StreamingRecognitionConfig;
import com.google.cloud.speech.v1.StreamingRecognitionResult;
import com.google.cloud.speech.v1.StreamingRecognizeRequest;
import com.google.cloud.speech.v1.StreamingRecognizeResponse;
import com.google.protobuf.ByteString;

import java.io.IOException;
import java.io.InputStream;
import java.util.ArrayList;
import java.util.Collections;
import java.util.Date;
import java.util.List;
import java.util.concurrent.TimeUnit;

import io.grpc.ManagedChannel;
import io.grpc.internal.DnsNameResolverProvider;
import io.grpc.okhttp.OkHttpChannelProvider;
import io.grpc.stub.StreamObserver;


public class SpeechAPI {

    public static final List<String> SCOPE = Collections.singletonList("https://www.googleapis.com/auth/cloud-platform");
    public static final String TAG = "SpeechAPI";

    private static final String PREFS = "SpeechService";
    private static final String PREF_ACCESS_TOKEN_VALUE = "access_token_value";
    private static final String PREF_ACCESS_TOKEN_EXPIRATION_TIME = "access_token_expiration_time";

    /**
     * We reuse an access token if its expiration time is longer than this.
     */
    private static final int ACCESS_TOKEN_EXPIRATION_TOLERANCE = 30 * 60 * 1000; // thirty minutes

    /**
     * We refresh the current access token before it expires.
     */
    private static final int ACCESS_TOKEN_FETCH_MARGIN = 60 * 1000; // one minute

    private static final String HOSTNAME = "speech.googleapis.com";
    private static final int PORT = 443;
    private static Handler mHandler;

    private final ArrayList<Listener> mListeners = new ArrayList<>();

    private final StreamObserver<StreamingRecognizeResponse> mResponseObserver = new StreamObserver<StreamingRecognizeResponse>() {
        @Override
        public void onNext(StreamingRecognizeResponse response) {
            String text = null;
            boolean isFinal = false;
            if (response.getResultsCount() > 0) {
                final StreamingRecognitionResult result = response.getResults(0);
                isFinal = result.getIsFinal();
                if (result.getAlternativesCount() > 0) {
                    final SpeechRecognitionAlternative alternative = result.getAlternatives(0);
                    text = alternative.getTranscript();
                }
            }
            if (text != null) {
                for (Listener listener : mListeners) {
                    listener.onSpeechRecognized(text, isFinal);
                }
            }
        }

        @Override
        public void onError(Throwable t) {
            Log.e(TAG, "Error calling the API.", t);
        }

        @Override
        public void onCompleted() {
            Log.i(TAG, "API completed.");
        }

    };
    private Context mContext;
    private volatile AccessTokenTask mAccessTokenTask;
    private final Runnable mFetchAccessTokenRunnable = new Runnable() {
        @Override
        public void run() {
            fetchAccessToken();
        }
    };
    private SpeechGrpc.SpeechStub mApi;
    private StreamObserver<StreamingRecognizeRequest> mRequestObserver;

    public SpeechAPI(Context mContext) {
        this.mContext = mContext;
        mHandler = new Handler();
        fetchAccessToken();

    }

    public void destroy() {
        mHandler.removeCallbacks(mFetchAccessTokenRunnable);
        mHandler = null;
        // Release the gRPC channel.
        if (mApi != null) {
            final ManagedChannel channel = (ManagedChannel) mApi.getChannel();
            if (channel != null && !channel.isShutdown()) {
                try {
                    channel.shutdown().awaitTermination(5, TimeUnit.SECONDS);
                } catch (InterruptedException e) {
                    Log.e(TAG, "Error shutting down the gRPC channel.", e);
                }
            }
            mApi = null;
        }
    }

    private void fetchAccessToken() {
        if (mAccessTokenTask != null) {
            return;
        }
        mAccessTokenTask = new AccessTokenTask();
        mAccessTokenTask.execute();
    }

    public void addListener(@NonNull Listener listener) {
        mListeners.add(listener);
    }

    public void removeListener(@NonNull Listener listener) {
        mListeners.remove(listener);
    }

    /**
     * Starts recognizing speech audio.
     *
     * @param sampleRate The sample rate of the audio.
     */
    public void startRecognizing(int sampleRate) {
        if (mApi == null) {
            Log.w(TAG, "API not ready. Ignoring the request.");
            return;
        }

        // Configure the API
        mRequestObserver = mApi.streamingRecognize(mResponseObserver);

        StreamingRecognitionConfig streamingConfig = StreamingRecognitionConfig.newBuilder()
                .setConfig(RecognitionConfig.newBuilder()
                        .setLanguageCode("en-US")
                        .setEncoding(RecognitionConfig.AudioEncoding.LINEAR16)
                        .setSampleRateHertz(sampleRate)
                        .build()
                )
                .setInterimResults(true)
                .setSingleUtterance(true)
                .build();

        StreamingRecognizeRequest streamingRecognizeRequest = StreamingRecognizeRequest.newBuilder().setStreamingConfig(streamingConfig).build();
        mRequestObserver.onNext(streamingRecognizeRequest);
    }

    /**
     * Recognizes the speech audio. This method should be called every time a chunk of byte buffer
     * is ready.
     *
     * @param data The audio data.
     * @param size The number of elements that are actually relevant in the {@code data}.
     */
    public void recognize(byte[] data, int size) {
        if (mRequestObserver == null) {
            return;
        }
        // Call the streaming recognition API
        mRequestObserver.onNext(StreamingRecognizeRequest.newBuilder()
                .setAudioContent(ByteString.copyFrom(data, 0, size))
                .build());
    }

    /**
     * Finishes recognizing speech audio.
     */
    public void finishRecognizing() {
        if (mRequestObserver == null) {
            return;
        }
        mRequestObserver.onCompleted();
        mRequestObserver = null;
    }

    public interface Listener {
        //Called when a new piece of text was recognized by the Speech API.
        void onSpeechRecognized(String text, boolean isFinal);
    }

    private class AccessTokenTask extends AsyncTask<Void, Void, AccessToken> {

        @Override
        protected AccessToken doInBackground(Void... voids) {

            final SharedPreferences prefs = mContext.getSharedPreferences(PREFS, Context.MODE_PRIVATE);
            String tokenValue = prefs.getString(PREF_ACCESS_TOKEN_VALUE, null);
            long expirationTime = prefs.getLong(PREF_ACCESS_TOKEN_EXPIRATION_TIME, -1);

            // Check if the current token is still valid for a while
            if (tokenValue != null && expirationTime > 0) {
                if (expirationTime > System.currentTimeMillis() + ACCESS_TOKEN_EXPIRATION_TOLERANCE) {
                    return new AccessToken(tokenValue, new Date(expirationTime));
                }
            }

            final InputStream stream = mContext.getResources().openRawResource(R.raw.credential);
            try {
                final GoogleCredentials credentials = GoogleCredentials.fromStream(stream).createScoped(SCOPE);
                final AccessToken token = credentials.refreshAccessToken();
                prefs.edit()
                        .putString(PREF_ACCESS_TOKEN_VALUE, token.getTokenValue())
                        .putLong(PREF_ACCESS_TOKEN_EXPIRATION_TIME, token.getExpirationTime().getTime())
                        .apply();
                return token;
            } catch (IOException e) {
                Log.e(TAG, "Failed to obtain access token.", e);
            }
            return null;
        }

        @Override
        protected void onPostExecute(AccessToken accessToken) {
            mAccessTokenTask = null;
            final ManagedChannel channel = new OkHttpChannelProvider()
                    .builderForAddress(HOSTNAME, PORT)
                    .nameResolverFactory(new DnsNameResolverProvider())
                    .intercept(new GoogleCredentialsInterceptor(new GoogleCredentials(accessToken)
                            .createScoped(SCOPE)))
                    .build();
            mApi = SpeechGrpc.newStub(channel);

            // Schedule access token refresh before it expires
            if (mHandler != null) {
                mHandler.postDelayed(mFetchAccessTokenRunnable,
                        Math.max(accessToken.getExpirationTime().getTime() - System.currentTimeMillis() - ACCESS_TOKEN_FETCH_MARGIN, ACCESS_TOKEN_EXPIRATION_TOLERANCE));
            }
        }
    }
}
```

6.Update the Project Gradle
```
// Top-level build file where you can add configuration options common to all sub-projects/modules.

buildscript {
    repositories {
        jcenter()
        maven {
            url 'https://maven.google.com/'
            name 'Google'
        }
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.3.3'
        classpath 'com.google.protobuf:protobuf-gradle-plugin:0.8.0'

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

allprojects {
    repositories {
        jcenter()
        maven {
            url 'https://maven.google.com/'
            name 'Google'
        }
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}
```


7.Udate the App Gradle

```
apply plugin: 'com.android.application'
apply plugin: 'com.google.protobuf'

ext {
    grpcVersion = '1.4.0'
}
android {
    compileSdkVersion 26
    buildToolsVersion "26.0.0"
    defaultConfig {
        applicationId "[YOUR APPLICATION ID]"
        minSdkVersion 21
        targetSdkVersion 26
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"

        multiDexEnabled  true
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
    configurations.all {
        resolutionStrategy.force 'com.google.code.findbugs:jsr305:3.0.2'

    }
}
protobuf {
    protoc {
        artifact = 'com.google.protobuf:protoc:3.3.0'
    }
    plugins {
        javalite {
            artifact = "com.google.protobuf:protoc-gen-javalite:3.0.0"
        }
        grpc {
            artifact = "io.grpc:protoc-gen-grpc-java:${grpcVersion}"
        }
    }
    generateProtoTasks {
        all().each { task ->
            task.plugins {
                javalite {}
                grpc {
                    // Options added to --grpc_out
                    option 'lite'
                }
            }
        }
    }
}


dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    androidTestCompile('com.android.support.test.espresso:espresso-core:2.2.2', {
        exclude group: 'com.android.support', module: 'support-annotations'
    })
    compile 'com.android.support:appcompat-v7:26.1.0'
    compile 'com.android.support.constraint:constraint-layout:1.0.2'
    testCompile 'junit:junit:4.12'

    compile 'com.jakewharton:butterknife:8.7.0'
    annotationProcessor 'com.jakewharton:butterknife-compiler:8.7.0'

    compile 'com.android.support:design:26.1.0'


    // gRPC
    compile "io.grpc:grpc-okhttp:$grpcVersion"
    compile "io.grpc:grpc-protobuf-lite:$grpcVersion"
    compile "io.grpc:grpc-stub:$grpcVersion"
    compile 'javax.annotation:javax.annotation-api:1.2'
    protobuf 'com.google.protobuf:protobuf-java:3.3.1'

    compile group: 'com.google.api.grpc', name: 'grpc-google-cloud-speech-v1', version: '0.1.13'

    // OAuth2 for Google API
    compile('com.google.auth:google-auth-library-oauth2-http:0.7.0') {
        exclude module: 'httpclient'
    }

    compile 'com.android.support:multidex:1.0.0'
    compile 'com.android.support:design:26.+'

}

```

8.Add audio and internet permissions to the manifest

    <uses-permission android:name="android.permission.RECORD_AUDIO" />
    <uses-permission android:name="android.permission.INTERNET" />
    
9.Add colors.xml

```
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <color name="colorPrimary">#3F51B5</color>
    <color name="colorPrimaryDark">#303F9F</color>
    <color name="colorAccent">#FF4081</color>
    <color name="ABSOLUTEZERO">#0048BA</color>
    <color name="ACID">#B0BF1A</color>
    <color name="AERO">#7cb9e8</color>
    <color name="AIRSUPERIORITYBLUE">#72A0C1</color>
    <color name="ALABASTER">#EDEAE0</color>
    <color name="ALICEBLUE">#F0F8FF</color>
    <color name="ALLOYORANGE">#C46210</color>
    <color name="ALMOND">#EFDECD</color>
    <color name="AMBER">#FFBF00</color>
    <color name="AMETHYST">#9966CC</color>
    <color name="ANTIFLASHWHITE">#F2F3F4</color>
    <color name="ANTIQUEBRASS">#CD9575</color>
    <color name="ANTIQUEBRONZE">#665D1E</color>
    <color name="ANTIQUEFUCHSIA">#915C83</color>
    <color name="ANTIQUERUBY">#841B2D</color>
    <color name="ANTIQUEWHITE">#FAEBD7</color>
    <color name="AO">#008000</color>
    <color name="APPLEGREEN">#8DB600</color>
    <color name="APRICOT">#FBCEB1</color>
    <color name="AQUA">#00FFFF</color>
    <color name="AQUAMARINE">#7FFFD4</color>
    <color name="ARCTICLIME">#D0FF14</color>
    <color name="ARMYGREEN">#4B5320</color>
    <color name="ARTICHOKE">#8F9779</color>
    <color name="ARYLIDEYELLOW">#E9D66B</color>
    <color name="ASHGRAY">#B2BEB5</color>
    <color name="ASPARAGUS">#87A96B</color>
    <color name="ATOMICTANGERINE">#FF9966</color>
    <color name="AUBURN">#A52A2A</color>
    <color name="AUREOLIN">#FDEE00</color>
    <color name="AVOCADO">#568203</color>
    <color name="AZURE">#007FFF</color>
    <color name="BABYBLUE">#89CFF0</color>
    <color name="BABYBLUEEYES">#A1CAF1</color>
    <color name="BABYPINK">#F4C2C2</color>
    <color name="BAKERMILLERPINK">#FF91AF</color>
    <color name="BANANAMANIA">#FAE7B5</color>
    <color name="BARBIEPINK">#E0218A</color>
    <color name="BARNRED">#7C0A02</color>
    <color name="BATTLESHIPGRAY">#848482</color>
    <color name="BDAZZLEDBLUE">#2E5894</color>
    <color name="BEAUBLUE">#BCD4E6</color>
    <color name="BEAVER">#9F8170</color>
    <color name="BEIGE">#F5F5DC</color>
    <color name="BIGDIPOsRUBY">#9C2542</color>
    <color name="BISQUE">#FFE4C4</color>
    <color name="BISTRE">#3D2B1F</color>
    <color name="BISTREBROWN">#967117</color>
    <color name="BITTERLEMON">#CAE00D</color>
    <color name="BITTERLIME">#BFFF00</color>
    <color name="BITTERSWEET">#FE6F5E</color>
    <color name="BITTERSWEETSHIMMER">#BF4F51</color>
    <color name="BLACK">#000000</color>
    <color name="BLACKBEAN">#3D0C02</color>
    <color name="BLACKCHOCOLATE">#1B1811</color>
    <color name="BLACKCOFFEE">#3B2F2F</color>
    <color name="BLACKCORAL">#54626F</color>
    <color name="BLACKOLIVE">#3B3C36</color>
    <color name="BLACKSHADOWS">#BFAFB2</color>
    <color name="BLANCHEDALMOND">#FFEBCD</color>
    <color name="BLASTOFFBRONZE">#A57164</color>
    <color name="BLEUDEFRANCE">#318CE7</color>
    <color name="BLIZZARDBLUE">#ACE5EE</color>
    <color name="BLOND">#FAF0BE</color>
    <color name="BLOODRED">#660000</color>
    <color name="BLUE">#0000FF</color>
    <color name="BLUEBELL">#A2A2D0</color>
    <color name="BLUEGRAY">#6699CC</color>
    <color name="BLUEGREEN">#064E40</color>
    <color name="BLUEJEANS">#5DADEC</color>
    <color name="BLUESAPPHIRE">#126180</color>
    <color name="BLUETIFUL">#3C69E7</color>
    <color name="BLUEVIOLET">#4D1A7F</color>
    <color name="BLUEYONDER">#5072A7</color>
    <color name="BLUSH">#DE5D83</color>
    <color name="BOLE">#79443B</color>
    <color name="BONDIBLUE">#0095B6</color>
    <color name="BONE">#E3DAC9</color>
    <color name="BOTTLEGREEN">#006A4E</color>
    <color name="BRANDY">#87413F</color>
    <color name="BRICKRED">#CB4154</color>
    <color name="BRIGHTGREEN">#66FF00</color>
    <color name="BRIGHTLILAC">#D891EF</color>
    <color name="BRIGHTMAROON">#C32148</color>
    <color name="BRIGHTNAVYBLUE">#1974D2</color>
    <color name="BRIGHTPINK">#FF007F</color>
    <color name="BRIGHTYELLOW">#FFAA1D</color>
    <color name="BRILLIANTROSE">#FF55A3</color>
    <color name="BRINKPINK">#FB607F</color>
    <color name="BRITISHRACINGGREEN">#004225</color>
    <color name="BRONZE">#CD7F32</color>
    <color name="BROWN">#88540B</color>
    <color name="BROWNSUGAR">#AF6E4D</color>
    <color name="BRUNSWICKGREEN">#1B4D3E</color>
    <color name="BUBBLES">#E7FEFF</color>
    <color name="BUDGREEN">#7BB661</color>
    <color name="BUFF">#F0DC82</color>
    <color name="BURGUNDY">#800020</color>
    <color name="BURLYWOOD">#DEB887</color>
    <color name="BURNISHEDBROWN">#A17A74</color>
    <color name="BURNTORANGE">#CC5500</color>
    <color name="BURNTSIENNA">#E97451</color>
    <color name="BURNTUMBER">#8A3324</color>
    <color name="BYZANTINE">#BD33A4</color>
    <color name="BYZANTIUM">#702963</color>
    <color name="CADET">#536872</color>
    <color name="CADETBLUE">#5F9EA0</color>
    <color name="CADETGRAY">#91A3B0</color>
    <color name="CADMIUMGREEN">#006B3C</color>
    <color name="CADMIUMORANGE">#ED872D</color>
    <color name="CADMIUMRED">#E30022</color>
    <color name="CADMIUMYELLOW">#FFF600</color>
    <color name="CAFÉAULAIT">#A67B5B</color>
    <color name="CAFÉNOIR">#4B3621</color>
    <color name="CALPOLYPOMONAGREEN">#1E4D2B</color>
    <color name="CAMBRIDGEBLUE">#A3C1AD</color>
    <color name="CAMEL">#C19A6B</color>
    <color name="CAMEOPINK">#EFBBCC</color>
    <color name="CANARY">#FFFF99</color>
    <color name="CANARYYELLOW">#FFEF00</color>
    <color name="CANDYAPPLERED">#FF0800</color>
    <color name="CANDYPINK">#E4717A</color>
    <color name="CAPRI">#00BFFF</color>
    <color name="CAPUTMORTUUM">#592720</color>
    <color name="CARDINAL">#C41E3A</color>
    <color name="CARIBBEANGREEN">#00CC99</color>
    <color name="CARMINE">#960018</color>
    <color name="CARNATIONPINK">#FFA6C9</color>
    <color name="CARNELIAN">#B31B1B</color>
    <color name="CAROLINABLUE">#56A0D3</color>
    <color name="CARROTORANGE">#ED9121</color>
    <color name="CASTLETONGREEN">#00563F</color>
    <color name="CATAWBA">#703642</color>
    <color name="CEDARCHEST">#C95A49</color>
    <color name="CELADON">#ACE1AF</color>
    <color name="CELADONBLUE">#007BA7</color>
    <color name="CELADONGREEN">#2F847C</color>
    <color name="CELESTE">#B2FFFF</color>
    <color name="CELTICBLUE">#246BCE</color>
    <color name="CERISE">#DE3163</color>
    <color name="CERULEAN">#007BA7</color>
    <color name="CERULEANBLUE">#2A52BE</color>
    <color name="CERULEANFROST">#6D9BC3</color>
    <color name="CGBLUE">#007AA5</color>
    <color name="CGRED">#E03C31</color>
    <color name="CHAMPAGNE">#F7E7CE</color>
    <color name="CHAMPAGNEPINK">#F1DDCF</color>
    <color name="CHARCOAL">#36454F</color>
    <color name="CHARLESTONGREEN">#232B2B</color>
    <color name="CHARMPINK">#E68FAC</color>
    <color name="CHARTREUSE">#7FFF00</color>
    <color name="CHERRYBLOSSOMPINK">#FFB7C5</color>
    <color name="CHESTNUT">#954535</color>
    <color name="CHINAPINK">#DE6FA1</color>
    <color name="CHINAROSE">#A8516E</color>
    <color name="CHINESERED">#AA381E</color>
    <color name="CHINESEVIOLET">#856088</color>
    <color name="CHINESEYELLOW">#FFB200</color>
    <color name="CHOCOLATE">#7B3F00</color>
    <color name="CHROMEYELLOW">#FFA700</color>
    <color name="CINEREOUS">#98817B</color>
    <color name="CINNABAR">#E34234</color>
    <color name="CINNAMONSATIN">#CD607E</color>
    <color name="CITRINE">#E4D00A</color>
    <color name="CITRON">#9FA91F</color>
    <color name="CLARET">#7F1734</color>
    <color name="COBALTBLUE">#0047AB</color>
    <color name="COCOABROWN">#D2691E</color>
    <color name="COFFEE">#6F4E37</color>
    <color name="COLUMBIABLUE">#B9D9EB</color>
    <color name="CONGOPINK">#F88379</color>
    <color name="COOLGRAY">#8C92AC</color>
    <color name="COPPER">#B87333</color>
    <color name="COPPERPENNY">#AD6F69</color>
    <color name="COPPERRED">#CB6D51</color>
    <color name="COPPERROSE">#996666</color>
    <color name="COQUELICOT">#FF3800</color>
    <color name="CORAL">#FF7F50</color>
    <color name="CORALPINK">#F88379</color>
    <color name="CORDOVAN">#893F45</color>
    <color name="CORN">#FBEC5D</color>
    <color name="CORNFLOWERBLUE">#6495ED</color>
    <color name="CORNSILK">#FFF8DC</color>
    <color name="COSMICCOBALT">#2E2D88</color>
    <color name="COSMICLATTE">#FFF8E7</color>
    <color name="COSMOSPINK">#FEBCFF</color>
    <color name="COTTONCANDY">#FFBCD9</color>
    <color name="COYOTEBROWN">#81613C</color>
    <color name="CREAM">#FFFDD0</color>
    <color name="CRIMSON">#DC143C</color>
    <color name="CULTURED">#F5F5F5</color>
    <color name="CYAN">#00B7EB</color>
    <color name="CYBERGRAPE">#58427C</color>
    <color name="CYBERYELLOW">#FFD300</color>
    <color name="CYCLAMEN">#F56FA1</color>
    <color name="DARKBLUE">#00008B</color>
    <color name="DARKBLUEGRAY">#666699</color>
    <color name="DARKBROWN">#654321</color>
    <color name="DARKBYZANTIUM">#5D3954</color>
    <color name="DARKCORNFLOWERBLUE">#26428B</color>
    <color name="DARKCYAN">#008B8B</color>
    <color name="DARKELECTRICBLUE">#536878</color>
    <color name="DARKGOLDENROD">#B8860B</color>
    <color name="DARKGRAY">#A9A9A9</color>
    <color name="DARKGREEN">#006400</color>
    <color name="DARKJUNGLEGREEN">#1A2421</color>
    <color name="DARKKHAKI">#BDB76B</color>
    <color name="DARKLAVA">#483C32</color>
    <color name="DARKLIVER">#534B4F</color>
    <color name="DARKMAGENTA">#8B008B</color>
    <color name="DARKMEDIUMGRAY">#A9A9A9</color>
    <color name="DARKMOSSGREEN">#4A5D23</color>
    <color name="DARKOLIVEGREEN">#556B2F</color>
    <color name="DARKORANGE">#FF8C00</color>
    <color name="DARKORCHID">#9932CC</color>
    <color name="DARKPASTELGREEN">#03C03C</color>
    <color name="DARKPURPLE">#301934</color>
    <color name="DARKRED">#8B0000</color>
    <color name="DARKSALMON">#E9967A</color>
    <color name="DARKSEAGREEN">#8FBC8F</color>
    <color name="DARKSIENNA">#3C1414</color>
    <color name="DARKSKYBLUE">#8CBED6</color>
    <color name="DARKSLATEBLUE">#483D8B</color>
    <color name="DARKSLATEGRAY">#2F4F4F</color>
    <color name="DARKSPRINGGREEN">#177245</color>
    <color name="DARKTURQUOISE">#00CED1</color>
    <color name="DARKVIOLET">#9400D3</color>
    <color name="DARTMOUTHGREEN">#00703C</color>
    <color name="DAVYSGRAY">#555555</color>
    <color name="DEEPCERISE">#DA3287</color>
    <color name="DEEPCHAMPAGNE">#FAD6A5</color>
    <color name="DEEPCHESTNUT">#B94E48</color>
    <color name="DEEPFUCHSIA">#C154C1</color>
    <color name="DEEPJUNGLEGREEN">#004B49</color>
    <color name="DEEPPINK">#FF1493</color>
    <color name="DEEPSAFFRON">#FF9933</color>
    <color name="DEEPSKYBLUE">#00BFFF</color>
    <color name="DEEPSPACESPARKLE">#4A646C</color>
    <color name="DEEPTAUPE">#7E5E60</color>
    <color name="DENIM">#1560BD</color>
    <color name="DENIMBLUE">#2243B6</color>
    <color name="DESERT">#C19A6B</color>
    <color name="DESERTSAND">#EDC9AF</color>
    <color name="DIMGRAY">#696969</color>
    <color name="DODGERBLUE">#1E90FF</color>
    <color name="DOGWOODROSE">#D71868</color>
    <color name="DRAB">#967117</color>
    <color name="DUKEBLUE">#00009C</color>
    <color name="DUTCHWHITE">#EFDFBB</color>
    <color name="EARTHYELLOW">#E1A95F</color>
    <color name="EBONY">#555D50</color>
    <color name="ECRU">#C2B280</color>
    <color name="EERIEBLACK">#1B1B1B</color>
    <color name="EGGPLANT">#614051</color>
    <color name="EGGSHELL">#F0EAD6</color>
    <color name="EGYPTIANBLUE">#1034A6</color>
    <color name="ELECTRICBLUE">#7DF9FF</color>
    <color name="ELECTRICGREEN">#00FF00</color>
    <color name="ELECTRICINDIGO">#6F00FF</color>
    <color name="ELECTRICLIME">#CCFF00</color>
    <color name="ELECTRICPURPLE">#BF00FF</color>
    <color name="ELECTRICVIOLET">#8F00FF</color>
    <color name="EMERALD">#50C878</color>
    <color name="EMINENCE">#6C3082</color>
    <color name="ENGLISHGREEN">#1B4D3E</color>
    <color name="ENGLISHLAVENDER">#B48395</color>
    <color name="ENGLISHRED">#AB4B52</color>
    <color name="ENGLISHVERMILLION">#CC474B</color>
    <color name="ENGLISHVIOLET">#563C5C</color>
    <color name="ERIN">#00FF40</color>
    <color name="ETONBLUE">#96C8A2</color>
    <color name="FALLOW">#C19A6B</color>
    <color name="FALURED">#801818</color>
    <color name="FANDANGO">#B53389</color>
    <color name="FANDANGOPINK">#DE5285</color>
    <color name="FASHIONFUCHSIA">#F400A1</color>
    <color name="FAWN">#E5AA70</color>
    <color name="FELDGRAU">#4D5D53</color>
    <color name="FERNGREEN">#4F7942</color>
    <color name="FIELDDRAB">#6C541E</color>
    <color name="FIERYROSE">#FF5470</color>
    <color name="FIREBRICK">#B22222</color>
    <color name="FIREENGINERED">#CE2029</color>
    <color name="FIREOPAL">#E95C4B</color>
    <color name="FLAME">#E25822</color>
    <color name="FLAVESCENT">#F7E98E</color>
    <color name="FLAX">#EEDC82</color>
    <color name="FLESH">#FFE9D1</color>
    <color name="FLIRT">#A2006D</color>
    <color name="FLORALWHITE">#FFFAF0</color>
    <color name="FLUORESCENTBLUE">#15F4EE</color>
    <color name="FORESTGREEN">#014421</color>
    <color name="FRENCHBEIGE">#A67B5B</color>
    <color name="FRENCHBISTRE">#856D4D</color>
    <color name="FRENCHBLUE">#0072BB</color>
    <color name="FRENCHFUCHSIA">#FD3F92</color>
    <color name="FRENCHLILAC">#86608E</color>
    <color name="FRENCHLIME">#9EFD38</color>
    <color name="FRENCHMAUVE">#D473D4</color>
    <color name="FRENCHPINK">#FD6C9E</color>
    <color name="FRENCHRASPBERRY">#C72C48</color>
    <color name="FRENCHROSE">#F64A8A</color>
    <color name="FRENCHSKYBLUE">#77B5FE</color>
    <color name="FRENCHVIOLET">#8806CE</color>
    <color name="FROSTBITE">#E936A7</color>
    <color name="FUCHSIA">#C154C1</color>
    <color name="FUCHSIAPURPLE">#CC397B</color>
    <color name="FUCHSIAROSE">#C74375</color>
    <color name="FULVOUS">#E48400</color>
    <color name="FUZZYWUZZY">#CC6666</color>
    <color name="GAINSBORO">#DCDCDC</color>
    <color name="GHOSTWHITE">#F8F8FF</color>
    <color name="GOLD">#FFD700</color>
    <color name="GOLDENROD">#DAA520</color>
    <color name="GRAY">#808080</color>
    <color name="GREEN">#008000</color>
    <color name="GREENYELLOW">#ADFF2F</color>
    <color name="HONEYDEW">#F0FFF0</color>
    <color name="HOTPINK">#FF69B4</color>
    <color name="INDIANRED">#CD5C5C</color>
    <color name="INDIGO">#4B0082</color>
    <color name="IVORY">#FFFFF0</color>
    <color name="KHAKI">#F0E68C</color>
    <color name="LAVENDER">#E6E6FA</color>
    <color name="LAVENDERBLUSH">#FFF0F5</color>
    <color name="LAWNGREEN">#7CFC00</color>
    <color name="LEMONCHIFFON">#FFFACD</color>
    <color name="LIGHTBLUE">#ADD8E6</color>
    <color name="LIGHTCORAL">#F08080</color>
    <color name="LIGHTCYAN">#E0FFFF</color>
    <color name="LIGHTGOLDENRODYELLOW">#FAFAD2</color>
    <color name="LIGHTGRAY">#D3D3D3</color>
    <color name="LIGHTGREEN">#90EE90</color>
    <color name="LIGHTPINK">#FFB6C1</color>
    <color name="LIGHTSALMON">#FFA07A</color>
    <color name="LIGHTSEAGREEN">#20B2AA</color>
    <color name="LIGHTSKYBLUE">#87CEFA</color>
    <color name="LIGHTSLATEGRAY">#778899</color>
    <color name="LIGHTSTEELBLUE">#B0C4DE</color>
    <color name="LIGHTYELLOW">#FFFFE0</color>
    <color name="LIME">#00FF00</color>
    <color name="LIMEGREEN">#32CD32</color>
    <color name="LINEN">#FAF0E6</color>
    <color name="MAGENTA">#FF00FF</color>
    <color name="MAROON">#800000</color>
    <color name="MEDIUMAQUAMARINE">#66CDAA</color>
    <color name="MEDIUMBLUE">#0000CD</color>
    <color name="MEDIUMORCHID">#BA55D3</color>
    <color name="MEDIUMPURPLE">#9370DB</color>
    <color name="MEDIUMSEAGREEN">#3CB371</color>
    <color name="MEDIUMSLATEBLUE">#7B68EE</color>
    <color name="MEDIUMSPRINGGREEN">#00FA9A</color>
    <color name="MEDIUMTURQUOISE">#48D1CC</color>
    <color name="MEDIUMVIOLETRED">#C71585</color>
    <color name="MIDNIGHTBLUE">#191970</color>
    <color name="MINTCREAM">#F5FFFA</color>
    <color name="MISTYROSE">#FFE4E1</color>
    <color name="MOCCASIN">#FFE4B5</color>
    <color name="NAVAJOWHITE">#FFDEAD</color>
    <color name="NAVY">#000080</color>
    <color name="OLDLACE">#FDF5E6</color>
    <color name="OLIVE">#808000</color>
    <color name="OLIVEDRAB">#6B8E23</color>
    <color name="ORANGE">#FFA500</color>
    <color name="ORANGERED">#FF4500</color>
    <color name="ORCHID">#DA70D6</color>
    <color name="PALEGOLDENROD">#EEE8AA</color>
    <color name="PALEGREEN">#98FB98</color>
    <color name="PALETURQUOISE">#AFEEEE</color>
    <color name="PALEVIOLETRED">#DB7093</color>
    <color name="PAPAYAWHIP">#FFEFD5</color>
    <color name="PEACHPUFF">#FFDAB9</color>
    <color name="PERU">#CD853F</color>
    <color name="PINK">#FFC0CB</color>
    <color name="PLUM">#DDA0DD</color>
    <color name="POWDERBLUE">#B0E0E6</color>
    <color name="PURPLE">#800080</color>
    <color name="REBECCAPURPLE">#663399</color>
    <color name="RED">#FF0000</color>
    <color name="ROSYBROWN">#BC8F8F</color>
    <color name="ROYALBLUE">#4169E1</color>
    <color name="SADDLEBROWN">#8B4513</color>
    <color name="SALMON">#FA8072</color>
    <color name="SANDYBROWN">#F4A460</color>
    <color name="SEAGREEN">#2E8B57</color>
    <color name="SEASHELL">#FFF5EE</color>
    <color name="SIENNA">#A0522D</color>
    <color name="SILVER">#C0C0C0</color>
    <color name="SKYBLUE">#87CEEB</color>
    <color name="SLATEBLUE">#6A5ACD</color>
    <color name="SLATEGRAY">#708090</color>
    <color name="SNOW">#FFFAFA</color>
    <color name="SPRINGGREEN">#00FF7F</color>
    <color name="STEELBLUE">#4682B4</color>
    <color name="TAN">#D2B48C</color>
    <color name="TEAL">#008080</color>
    <color name="THISTLE">#D8BFD8</color>
    <color name="TOMATO">#FF6347</color>
    <color name="TURQUOISE">#40E0D0</color>
    <color name="VIOLET">#EE82EE</color>
    <color name="WHEAT">#F5DEB3</color>
    <color name="WHITE">#FFFFFF</color>
    <color name="WHITESMOKE">#F5F5F5</color>
    <color name="YELLOW">#FFFF00</color>
    <color name="YELLOWGREEN">#9ACD32</color>
</resources>
```

    
## Google Speech API

We're going to need to set up our google cloud and retrieve an API key.

Go to the Google Cloud Platform [console](https://console.cloud.google.com).

1.Create a new Project
![Create](cproj.png)

2.Enable API
![Enable](cenable.png)

3.Add Cloud Speech API
![Add API](cspeech.png)

4.Create credentials
![Credentials](ccred.png)

5.Assign role
![Role](crole.png)

One you have done this it will generate and download a JSON file. Rename this file to credentials.json and then create a new directory in your project res file called raw and add the json file there.

![Raw](rawfile.png)



## Adding Functionality

Now it's time to start adding the functionality to change the background color!

In the `content_main.xml`, let's give the constraint layout a name, so we can reference it using `layout = (ConstraintLayout) findViewById(R.id.layout);`. Also feel free to get rid of the "Hello World" text.

Now let's go into the `activity_main.xml` and change the icon of the Floating Action Button to a mic. Find srcCompat in the properties view and change it to `ic_btn_speak_now`. Let's change the backgroundTint of the FAB to red, and the tint to WHITE.


![Layout](sc.png)


Now eventhough we have uses-permission in our manifest, we should request audio on runtime. Let's add this to our `onCreate()` method.
        
```
if (ContextCompat.checkSelfPermission(this, Manifest.permission.RECORD_AUDIO) != PackageManager.PERMISSION_GRANTED) {
	ActivityCompat.requestPermissions(this, new String[]{Manifest.permission.RECORD_AUDIO}, RECORD_REQUEST_CODE);
}
```
Putting our request code as a constant at the top of our class.

`private static final int RECORD_REQUEST_CODE = 101;`


Now we're going to set up our voice recorder.

```
private final VoiceRecorder.Callback mVoiceCallback = new VoiceRecorder.Callback() {

        @Override
        public void onVoiceStart() {
            if (speechAPI != null) {
                speechAPI.startRecognizing(mVoiceRecorder.getSampleRate());
            }
        }

        @Override
        public void onVoice(byte[] data, int size) {
            if (speechAPI != null) {
                speechAPI.recognize(data, size);
            }
        }

        @Override
        public void onVoiceEnd() {
            if (speechAPI != null) {
                speechAPI.finishRecognizing();
            }
        }

    };
```

Next, let's set up our SpeechAPI.

    private final SpeechAPI.Listener mSpeechServiceListener =
            new SpeechAPI.Listener() {
                @Override
                public void onSpeechRecognized(final String text, final boolean isFinal) {
                    processText(text);
                    if (isFinal) {
                        mVoiceRecorder.dismiss();
                    }
                }
            };

We need to process the text. So let's create that processText function.

    private void processText(final String text) {
        runOnUiThread(new Runnable() {
            @Override
            public void run() {
                String newText = text.replaceAll("\\s+","");
                layout.setBackgroundResource(getColorByName(newText.toUpperCase()));
            }
        });
    }
    
    
Now we need to get the color by name from our colors.xml resource file.

    public int getColorByName(String name) {
        int colorId = 0;
        try {
            Class res = R.color.class;
            Field field = res.getField(name);
            colorId = field.getInt(null);
        } catch (Exception e) {
            //e.printStackTrace();
        }
        return colorId;
    }
    
Now let's override on stop.

    @Override
    protected void onStop() {
        stopVoiceRecorder();

        // Stop Cloud Speech API
        speechAPI.removeListener(mSpeechServiceListener);
        speechAPI.destroy();
        speechAPI = null;

        super.onStop();
    }
    
    
We'll have these functions `startVoiceRecorder()` and `stopVoiceRecorder()`.

    
        private void startVoiceRecorder() {
        if (mVoiceRecorder != null) {
            mVoiceRecorder.stop();
        }
        mVoiceRecorder = new VoiceRecorder(mVoiceCallback);
        mVoiceRecorder.start();
    }

    private void stopVoiceRecorder() {
        if (mVoiceRecorder != null) {
            mVoiceRecorder.stop();
            mVoiceRecorder = null;
        }
    }
    

It's finally time to put all these together. We're going to call these functions when our floating action button is pressed, and then stop when it's pressed again.

Let's create a public variable `boolean flag = true;` so we know if the button has been pressed our not.

Now we can get rid of the auto-generated snackbar in our FAB listener code and instead have:

```java
            public void onClick(View view) {
                if (flag){
                    ViewCompat.setBackgroundTintList(fab, ColorStateList.valueOf(Color.parseColor("GREEN")));
                    flag = false;
                    startVoiceRecorder();
                    speechAPI.addListener(mSpeechServiceListener);
                } else {
                    ViewCompat.setBackgroundTintList(fab, ColorStateList.valueOf(Color.parseColor("RED")));
                    flag = true;
                    speechAPI.removeListener(mSpeechServiceListener);
                    stopVoiceRecorder();
                }
            }
```


Now we're ready to run it! When we click the floating action button, it should turn green and we can speak the colors.

![Layout](bp.png)

## Summary and Resources
In this tutorial we created the Colorwaves app that uses Google Cloud Speech API to change the color of our background to whatever color the user has spoken.

To learn more about the Google Speech API, please visit this [page](https://cloud.google.com/speech-to-text/).


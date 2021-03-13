#### [Http](https://www.jianshu.com/p/e853bb7b8634)



#### [Okhttp](https://www.jianshu.com/p/e0f65a6281fe)

#### Retrofit

##### 注解

<img src="pic\aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzk0NDM2NS1lZTc0N2QxZTMzMWVkNWE0LnBuZz9pbWFnZU1vZ3IyL2F1dG8tb3JpZW50L3N0cmlwJTdDaW1hZ2VWaWV3Mi8yL3cvMTI0MA" alt="注解类型" style="zoom:67%;" />

- 网络请求方法

  1. `@GET`

     ```kotlin
     @GET("api/goods/getRegion")
     suspend fun getCityList(@Query("page") page: Int, @Query("pageSize") pageSize: Int):BaseResultData<CityListRes?>
     @GET("userFourmZan/{id}/{type}")
     suspend fun postNewsLike( @Path("id") id: Int, @Path("type") type: Int): BaseResultData<Any?>
     ```

     

  2. `@POST`

     ```kotlin
      @POST("api/user/bindMobile")
      suspend fun bindMobile(@Body request: HashMap<String, Any?>): BaseResultData<LoginRes?>
     ```

- 标记类

  1. `@FormUrlEncoded`：表单

     请求头：Content-Type: application/x-www-form-urlencoded

     ```java
     @FormUrlEncoded
     @POST("/")
     Call<ResponseBody> example(@Field("name") String name,@Field("occupation") String occupation);
     //foo.example("Bob Smith", "President")->name=Bob Smith&occupation=President
     
     @FormUrlEncoded
     @POST("/")
     Call<ResponseBody> example(@FieldMap Map<String,String> fields);
     //foo.things(ImmutableMap.of("foo", "bar", "kit", "kat")-> foo=bar&kit=kat
     ```
     
  2. `@Multipart`:上传文件
  
     请求头：Content-Type：multipart/form-data
  
     

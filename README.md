# milestone5
* `types/videosdk.d.ts`: khai báo (export) definition built sẵn của cái livestream sdk
* `components/auth/LoginFormCard.tsx`:
  thầy có hỏi cái svg dòng 30 là vẽ cái gì thì m bảo là vẽ logo Google bằng svg, mỗi path là 1 màu của logo Google
  dòng 2 dùng signIn, signOut của library next-auth là 1 lib của next.js
* `utils/livestream.ts`: này là file mới duy nhất
  * m xem mấy cái config của nó
  * brandLogoURL https://picsum.photos/200 là 1 cái pseudo image để tạm, mốt thêm brand app của mình
  * xong r tạo meeting với config như vậy r initiate cái meeting thôi
* m vào ***aws***
  * vào services -> compute -> lambda
  * vào ***e-learning-core-function***
  * vào file `classes-table.js`
```javascript
const AWS = require("aws-sdk"); // thay vì dùng import AWS from 'aws-sdk' thì dùng hàm cũ require
const dynamo = new AWS.DynamoDB.DocumentClient(); // tạo database dynamo
const classTable = "classes"; // tên table là classes

const getClass = async (classId) => {
    var params = {
        TableName: classTable,
        Key: {
            id: classId
        }
    };
    return dynamo.get(params).promise(); // lấy data, với id là classId, từ database
}

const getAllClasses = async () => {
    var params = {
        TableName: classTable
    }
    return dynamo.scan(params).promise(); // lấy tất cả class trong table "classes"
}

const getAllMessages = async () => {
    var params = {
        TableName: classTable
    }
    return dynamo.scan(params) // chưa làm send message, upload messages lên / lấy messages từ database về
}

const addClass = async (uuid, classObject) => {
    var params = {
        TableName: classTable,
        Item: {
            id: uuid,
            ...classObject
        }
    }
    return dynamo.put(params).promise(); // thêm class với id, messages, ... vào table "classes"
}

const deleteClass = async (classId) => {
    var params = {
        TableName: classTable,
        Key: {
            id: classId
        }
    }
    return dynamo.delete(params).promise(); // xoá class
}

module.exports = {
    getClass,
    getAllClasses,
    addClass,
    deleteClass
}; // thay vì dùng export {...} thì dùng cũ module.exports
```
* file users-table.js tương tự
* file index.js cần lưu ý
  * dùng `exports.handler` là quy ước của aws lambda để biết sẽ phải chạy hàm này, gần giống export bt v
  * có 2 statusCode: `200` là request lên server aws successfully, `400` là error
  * có 4 event là ***GET*** và ***POST*** để lấy data, ***PUT*** để set data (thêm class, user), ***DELETE*** để xoá data (xoá class)
  * vd `${userPath}/{id}` là lấy user với id từ database table `"users"`
  * vd `${classPath}/{id}/messages` là lấy tất cả message từ class với id từ database table `"classes"`
* ok giờ vào ***e-learning-upload-function***
```javascript
// Create the S3 service object

const AWS = require('aws-sdk');
AWS.config.update({
    region: 'ca-central-1',
    signatureVersion: 'v4'
});
const s3 = new AWS.S3(); // Tạo s3, giống như tạo account google drive để up file lên v

// Change this value to adjust the signed URL's expiration
const URL_EXPIRATION_SECONDS = 300; // Thời gian đổi signed URL, sau 300 giây signed URL cũ hết hiệu lực sẽ phải gọi lên server request signed URL mới

// Main Lambda entry point
exports.handler = async (event, context, callback) => {
    return await getUploadURL(event, context, callback); // Trả về cái response nằm ở dòng cuối của cái code ấy, trong đó có responseBody là signedURL (uploadURL) hoặc là error
};

const getUploadURL = async function (event, context, callback) {
    var statusCode = 500; // statusCode 500 là lỗi Internal Server Error
    var responseBody = {};
    
    console.log(event);
    try {
        let method = event.httpMethod;
        let path = event.path;
        // let resource = method + " " + path;
        let resource = event.routeKey;
        
        if (resource == "GET /upload") {
            if (!("queryStringParameters" in event))
                throw new Error("Query params is empty.");
            const queryParams = event.queryStringParameters; // Là cái request lấy signed URL gửi lên server sẽ bao gồm các param nằm trong event.queryStringParameters

            if ("fileType" in queryParams && 
                "dir" in queryParams) {
                const fileName = context.awsRequestId; // Tên file
                const fileType = queryParams.fileType.toLowerCase(); // Đuôi file thì lowercase nó (.jpg thay vì .JPG, .Jpg)
                const file = `${fileName}.${fileType}`;
                const dir = queryParams.dir;

                // Get signed URL from S3
                const s3Params = {
                    Bucket: process.env.UploadBucket,
                    Key: `${dir}/${file}`,
                    Expires: URL_EXPIRATION_SECONDS,
                    ContentType: fileType
    
                    // This ACL makes the uploaded object publicly readable. You must also uncomment
                    // the extra permission for the Lambda function in the SAM template.
                    // ACL: 'public-read'
                };
    
                console.log('Params: ', s3Params);
    
                const uploadURL = await s3.getSignedUrlPromise('putObject', s3Params); // Yêu cầu tạo signedURL để upload file nè
                responseBody = JSON.stringify({
                    uploadURL: uploadURL,
                    file: `${dir}/${file}`,
                    expiration: Math.floor(Date.now()/1000) + URL_EXPIRATION_SECONDS
                }); // Assign cái signedURL (uploadURL) vừa tạo vào responseBody để trả về xuống cho client
                statusCode = 200;   // success            
            } else {
                statusCode = 400;   // client error
                responseBody = {
                    "error": "Missing `dir` or `fileType` params."
                };
            }
        } else {
            throw new Error(`Unsupported route: "${event.routeKey}"`);
        }
    } catch (err) {
        statusCode = 500;   // server error
        responseBody = { "error": err };
        console.log(err)
    }

    const response = {
        statusCode: statusCode,
        headers: {
            "Access-Control-Allow-Headers": "Content-Type",
            "Access-Control-Allow-Origin": "*",
            "Access-Control-Allow-Methods": "OPTIONS,PUT,POST,GET",
            "Content-Type": "application/json",
        },
        body: responseBody,
    };

    return response; // Đây trả về cái response mà có body là cái responseBody chứa signedURL (uploadURL) hoặc là error
};
```

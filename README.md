# kong
**If you are reporting *any* crash or *any* potential security issue, *do not*
open an issue in this repo. Please report the issue via emailing
envoy-security@googlegroups.com where the issue will be triaged appropriately.**

*Title*: *One line description*
How to effectively modify the buffer in decodeData
*Description*:
I'm trying to edit buffer in DecodeData function of Go' filter.But I can't understand why It doesn't work sometimes.For example, the WriteString does not work if the code as below. But when I uncommented "buffer.Set(newData)", it took effect.
`
func (f *filter) DecodeData(buffer api.BufferInstance, endStream bool) api.StatusType {
        buffer.WriteString("Decodedata")
	t := time.Now()
	fmt.Println("time is", t.Format("2006-01-02 15:04:05.000"))
	if endStream == false {
		result := make(chan string)
		// 将data反序列化到map中
		reqInfo := make(map[string]interface{})
		err := json.Unmarshal([]byte(buffer.String()), &reqInfo)
		if err != nil {
			fmt.Println("Error:", err)
			return api.Continue
		}
		nums, ok := reqInfo["nums"].([]interface{})
		if ok {
			reqInfo["nums"] = append(nums, 10)
		}
		// 将map重新序列化为json
		newData, err := json.Marshal(reqInfo)
		//buffer.Set(newData)
		go f.post(string(newData), result, &buffer)
		res, ok := <-result
		if ok {
			if num, err := strconv.Atoi(res); err == nil {
				if num > 1000000 {
					f.callbacks.SendLocalReply(200, "result too large", map[string]string{}, 0, "too-large")
					return api.LocalReply
				}
			}
		}
	}
	fmt.Println("data is ", buffer.String())
	return api.Continue
}
`
the result of comment buffer.Set() is "**b'{"operation": "add", "nums": [1, 2, 3, 4, 5, 6, 9, 9]}|| response || encodedata append_data'**"
and the result of uncomment is "**b'{"nums":[1,2,3,4,5,6,9,9,10],"operation":"add"}Decoded|| response || encodedata append_data**'".
It shows that the WriteString and Set on buffer works. And When I am trying to increase the request body(set "nums":[1,2,3,4,5,6,9,9,10,11,12,13]),envoy response with "upstream request timeout". Here is part of this request's log of envoy:
[2023-09-08 12:12:46.121][17][debug][client] [source/common/http/codec_client.cc:139] [C5] encode complete
data is  {"nums":[1,2,3,4,5,6,9,9,10,11,12,13,10],"operation":"add"}
time is 2023-09-08 12:12:46.121
data is  
[2023-09-08 12:12:48.100][17][debug][router] [source/common/router/router.cc:1017] [C4][S7357123165049975762] upstream timeout
[2023-09-08 12:12:48.100][17][debug][router] [source/common/router/upstream_request.cc:557] [C4][S7357123165049975762] resetting pool request
[2023-09-08 12:12:48.100][17][debug][client] [source/common/http/codec_client.cc:156] [C5] request reset
[2023-09-08 12:12:48.100][17][debug][connection] [source/common/network/connection_impl.cc:139] [C5] closing data_to_write=0 type=1
[2023-09-08 12:12:48.100][17][debug][connection] [source/common/network/connection_impl.cc:250] [C5] closing socket: 1

So I am confused how can I edit the buffer effectively. I hope you can exa
[optional *Relevant Links*:]
This is my request code
`mydata={
    "operation":"add",
    "nums":[1,2,3,4,5,6,9,9,10,11,12,13]
}
udata = json.dumps(mydata)
response = requests.post("http://localhost:10000",data=udata)` @doujiang24 

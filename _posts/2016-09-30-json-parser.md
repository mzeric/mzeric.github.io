---
layout: post
title: 五脏俱全的 json parser
categories: compiler
---

json 文法简洁，正好用来演示 手工写parser的过程

{% highlight C %}
/*

Recursive-Decent Grammar Parser 4 Json

Author: tianxiao
Email: mzericxia@gmail.com
Github: @mzeric

*/
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>
#include <string>
#include <iostream>
/*

json_object :  { name_pair }

name_pair	:	name_value  |	name_pair , name_value

name_value	:	str	:  json_value

json_value	:	json_jobject |  json_array
			:	str | NUM | TRUE | FALSE | NULL

json_array	:	[ some_value ]

some_value	:	json_value   |	 some_value , json_value


*/

const char* input_str = "{ \"name\" : \"skysmiler\", \"data\":[\"value\"]}";
static char look_ahead;
char read_next() {
	static int pc = 0;
	return input_str[pc++];
}
void match_space(){
	do{
		look_ahead = read_next();
	}while(look_ahead == ' ' || look_ahead == '\t');
}
void match(char c) {
	printf("try match %c == %c\n", look_ahead, c);
	assert(look_ahead == c);
	match_space();
}
std::string match_str(){
	char buf[128];

	assert(look_ahead == '\"');
	memset(buf, 0, sizeof(buf));
	int i = 0;
	do {
		look_ahead = read_next();
		buf[i++] = look_ahead;
	}while(look_ahead != '\"');
	buf[i-1] = 0;
	match_space();

	return std::string(buf);
}
int match_bool(char c){
	int ret;
	assert(c == 't' || c == 'f');
	if(c == 't'){
		ret = 1;
	}else if(c == 'f'){
		ret = 0;
	}
	match_space();

	return ret;
}
double match_digit(){
	char buf[128];
	int i = 0;
	do {
		look_ahead = read_next();
		buf[i++] = look_ahead;
	}while((look_ahead <='9' && look_ahead >='0') || look_ahead=='.');
	buf[i] = 0;
	match_space();

	return atof(buf);
}
void name_value();
void name_pair();
void json_value();
void json_array();
void some_value();

void json_object () {
	match('{');name_pair(); match('}');
}
void name_pair_(){
	if(look_ahead == ',') {
		match(',');name_value();name_pair_();
	}else
	;
}
void name_pair() {
	name_value();name_pair_();
}
void name_value() {
	std::string name;
	name = match_str();match(':');json_value();
	std::cout << "Get Name: " << name << std::endl;
}
void json_value() {
	if(look_ahead == '{')
		json_object();
	else if(look_ahead == '[')
		json_array();
	else if(look_ahead == '\"'){
		std::string d = match_str();
		std::cout << "Get Value: " << d <<std::endl;
	}else{
		// number,true, false, null
		if(look_ahead == '\"')
			match_str();
		else if(look_ahead == 't' || look_ahead == 'f'){
			match_bool(look_ahead);
		}else if(look_ahead<='9' && look_ahead >='0'){
			match_digit();
		}else
		;
	}
}
void json_array() {
	match('['); some_value();match(']');
}
void some_value_() {
	if(look_ahead == ','){
		match(',');json_value();some_value_();
	}else
	;
}
void some_value() {
	json_value();some_value_();
}

int main(){
	look_ahead = read_next();

	json_object();
	printf("%s\n", input_str);
}


{% endhighlight %}

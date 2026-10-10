action = create
passing the instances names in list
1. create instances, loop instance names. item is the keyword
	register the output in variable -> ec2_output
	
	ec2_output.results -> output as list
	
2. loop ec2_output.results list of map
	item -> first map
	
	{{ item.item }} => name
	{{ item.instances[0].private_ip_address }}
	
	frontend-dev.daws88s.online -> private_ip_address
	
	roboshop-dev.daws88s.online -> public_ip_address
	loop: ec2_output.results
	when: 
	- item.item == "frontend"
	- action == "create"
	

	
	
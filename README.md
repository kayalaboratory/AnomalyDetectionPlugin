# AnomalyDetectionPlugin
Real-Time Anomaly Detection in Industry 4.0 Using Asset Administration Shell

*This repository describes how to integrate the Anomaly Detection Plugin into Eclipse BaSyx.*

Please first check the [Eclipse BaSyx GitHub Repository](https://github.com/eclipse-basyx/basyx-aas-web-ui) to perform installation.

Instructions for devoloping custom plugin for Eclipse BaSyx [can be found here.](https://wiki.basyx.org/en/latest/content/user_documentation/basyx_components/web_ui/features/plugin_mechanism.html)

To use the plugin the installation procedure should be finished. After cloning the repository from Eclipse BaSyx GitHub page, go into examples folder and build the docker file by executing docker-compose command.

![image](https://github.com/user-attachments/assets/37d0ddbb-0857-4b2e-b9a8-4af41f89ffc0)

When Docker is set up, these containers should be seen on the docker app:

![image](https://github.com/user-attachments/assets/eba3f90e-7646-42c5-ba2d-20326ffe59bd)

Download the AnomalyDetectionPlugin and move it inside aas-web-ui/src/UserPlugins folder.

![image](https://github.com/user-attachments/assets/9e4aec69-5b15-4af4-9a2a-08ae8cb9af6a)


Now start the containers except "aas-ui", we will start aas-ui from another source. For this go to basyx-aas-web-ui-main/aas-web-ui directory, run these commands:
```
yarn install
yarn dev
```

This will start aas-ui on the port 3000 by default. If this port is occupied it might be 3001 etc. When containers started the registry will take the AASX file inside basyx-aas-web-ui-main\examples\TimeSeriesData\aas folder by default, but AASX files can be imported from ui.

![image](https://github.com/user-attachments/assets/041a05e7-0cad-4bfa-896e-52272be387ae)

If imported AASX file has a submodel which has semantic id "http://submodel_template/AnomalyDetection" the plugin will be accessable automatically. A sample AASX file is provided in the repository that is modified version of Time-Series submodel given in the Eclipse BaSyx Github repository. For editing AASX file BaSyx UI or [AASX Package Explorer](https://github.com/eclipse-aaspe/package-explorer) can be used.



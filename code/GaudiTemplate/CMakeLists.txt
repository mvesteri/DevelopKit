cmake_minimum_required(VERSION 2.8.5)

find_package(GaudiProject)

# Set this to the name of the project you want to create
# (or just use the package in another project, in which
# case you should not need this file)

gaudi_project(GaudiTemplate v1r0

# This should be set to Gaudi, Lbcom, or similar
              USE Gaudi v28r0
              # USE Lbcom v19r1
# Any data dependencies
#             DATA MyData v*
             )

